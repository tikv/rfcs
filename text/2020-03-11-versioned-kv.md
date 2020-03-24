# Versioned KV (VerKV)

## Summary

This proposal presents another KV interface - Versioned KV (VerKV), a
multi-versioned KV between RawKV and TxnKV.

## Motivation

At present, Yidian Zixun adopts TiKV RawKV for Key-Value storage. Due to
performance concerns, TxnKV is not used. However, RawKV only provides simple KV
semantics, and does not support multiple versions based on timestamp,
which causes some problems:

- When services are migrated from HBase to TiKV, RawKV does not fulfil some
  major functionality requirements such as data expiration and historical data
   version;
- When developing backup and synchronization functions based on RawKV,
  it is difficult to solve the problems of online data migration, incremental
   backup, and data synchronization between two data centers because timestamp
  is not supported.

In order to solve the above problems, this proposal presents another kv
interface - VerKV, which mainly addsversion information such as timestamp to
the key. At the same time, this interface will be designed to be approximate
to the semantics of HBase, which will facilitate the service migration from
HBase to TiKV, thereby integrating TiKV better into the HBase ecosystem.

## Detailed designs

### Functions and scenarios

Here are the functionality designs and the corresponding application
scenarios for VerKV:

- Support data migration, incremental backup, CDC (Change Data Capture)
  and cross-IDC cluster synchronization application scenarios include:

  - Online data replication between TiKV clusters, which includes a full
    migrations followed by incremental migrations;
  - Cluster data backup, which includes a first full backup followed perform
    an incremental data backups every other time;
  - CDC service created based on incremental data acquisition; can reliably
    provide incremental data services;
  - In cross data center cluster synchronization scenarios, VerKV can not
    only process incremental data synchronization, but also handle write
    conflicts caused by the same key.

- Support multi-versioned data. The corresponding application scenarios
  include:

  - Data version retrieval. For certain businesses, multiple versions of
    historical data must be maintained for for backtracking and analysis.
    Reliance on third-party storage and consumption of computing resources
    (user dynamic characteristics, User behavior link tracking) needs to be
    reduced, and the data processing process optimized;
  - Data version control, for fast data recovery after problematic data
    writes in businesses.

- Support data expiration based on TTL. The corresponding application
  scenarios include:

  - Setting expiration time to obtain data of a recent period
  - Setting expiration time to clear inactive data for a specified amount
    of time

### Compatibility

The design adds a new Column Family called `VerCF` in RocksDB, which avoids
potential compatibility issues with the existing RawKV annd TxnKV data and
processing logics, and could be created dynamically when needed.
This design allows maximum reuse of the current implementations of TiKV and PD.

### Data model and interface

VerKV is a kv-like interface similar to the RawKV interface. It hides version
information from upper-layer services. Its data model is multi-versioned KV
between RawKV and TxnKV.

#### Data model and Consistency

The data model is like `key:version -> value`. The version uses the global `ts`
obtained from the PD TSO by default. Then strong consistency (single key level
 linearizability) would be support. And the implementation is subject to
 further change if we use the user TS feature of RocksDB.

The maximum number of versions in the cluster can be configured via `MaxVerNum`.
The default value is 1. Redundant versions are removed during version
recycling (compaction or garbage collection).

#### Data storage

`user_key` is encoded according to Memory Comparable Encoding in the TiKV client
sdk, and the encoding algorithm is padding in 8-byte units. This operation
 ensures that ts is appended to the key and the written user_key (key: ts)
can keep the same order as the key itself.

The timestamp stored in user_key is its inversion result. The purpose is
to arrange the newer data (that is, the data with larger timestamp) in front
 of the older data (that is, data with smaller timestamp).
This feature can be utilized in performance optimizing for the data
scan process.

#### Data distribution

To facilitate the acquisition and efficient management of multiple versions of
a key, the split point of the region will not use the timestamp of the key
, so that multiple versions of the key can be guaranteed to be
in the same region and the same CF. Considering the size of the region,
MaxVerNum will be restrained.

The process of data balancing, scaling, and migration are the same as that
of RawKV. In the process of scanning all versions of a key, if the maximum
 number of versions is exceeded, the redundant versions will be discarded.

#### RPC Interface

With reference to the proto interface, the following interfaces are defined as
a new service other than tikv:

- Get/BatchGet - Gets the values for latest version of the key by default
- GetNum/BatchGetNum - Get the values for a specified number of versions of a
  key. `version_num` must be less than or equal to `MaxVerNum`.
- GetNumWithSV/BatchGetNumWithSV - This interface is like `GetNum/BatchGetNum`,
  but add requirement about the key with versions ranging from the specified
  start version to the current version.
- GetWithTTL/BatchGetWithTTL - Gets the unexpired values within the TTL.
- Scan - Scans the values of all keys of the latest version within a range
- ScanNum - Scans the number of all keys within a range to get the
  corresponding values of a specified number of versions
- ScanNumWithSV - Scans all the keys within a range and gets the
  corresponding values ranging from the specified start version to the current
  version
- Put/BatchPut - Increases the new version value of a key, where version is
  the current timestamp by default
- Delete/BatchDelete - Deletes all versions of a specified key
  (`key:cur_ts -> delete`)
- DeleteRange - Deletes all versions of key in a range using rocksdb DeleteRange

The above fetch operations(get/scan) can use `rocksdb prefix_extractor` to
improve the performance of iterator seek. `key: ts-> delete` indicates that
 all data before the `ts` has been deleted.

### Version recycling

The version recycling would recycling the redundant versions more than
`MaxVerNum`, e.g. one key has five versions [v1 v2 v3 v4 v5], and the
`MaxVerNum` is set to be 3, then [v1 v2] would be recycled.

The design of the version recycling will affect cluster performance and the
functional designs of incremental backup and CDC. There are two major
approaches:

- using multi-version Garbage Collection (GC) similar to TxnKV, and
  periodically starting the gc worker in the background to delete redundant
  versions of data;
- using compaction filter via rocksdb compaction to delete redundant versions.
  The default maximum number of versions is 1, which can default to the
  compaction filter recycling version , so recycling efficiency is moderate.

The GC method consumes CPU or storage resources, but the version recycling is
accurate and efficient; the compaction filter method does not take up CPU or
storage resources. However, compaction only has a local applicable scope, so
multi-version recycling efficiency is low.

This proposal adopts GC method mainly, and could reuse the logic of TxnKV GC.

### Data increment

The `ScanNumWithSV` rpc interface is used to get all data increments of the
entire cluster or within a specified range [start_ts, cur_ts). When this
interface is called, the snapshot of the entire cluster or the current cur_ts
is obtained before scanning.

VerKV integrated to tidb/br and tidb/ticdc can better facilitate the
implementation of full/incremental backup and CDC services.

### Data replication

For data replication, there are two main situations:

- A -> B: asynchronous one-way replication, Cluster B accesses the CDC
  service of cluster A to pull or push incremental data of each period
- A <-> B: asynchronous two-way replication, which is equivalent to two
  one-way replication processes. Write conflicts can be resolved according to ts

### Others

Rocksdb User Timestamp（WIP）can also be considered to improve the engine
efficiency.

## Drawbacks

This VerKV adds version information, so it would decrease the performance
of data fetch interface. And the data version recycling would also add
pressure to system.
