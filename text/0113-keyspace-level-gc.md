# Keyspace level GC

- RFC PR: https://github.com/tikv/rfcs/pull/113
- Tracking Issue: https://github.com/tikv/tikv/issues/16896

## Summary

TiKV support [keyspace][1] level GC (garbage collection).

## Concepts of GC management type

1. Global GC:
   - It represents the previous default GC logic, where TiDB calculates the global GC safe point for the entire cluster.
   - The default GC management type for keyspace is the Global GC.
2. Keyspace Level GC:
   - It indicates that the keyspace will advance its own GC safe point.
   - Keyspace GC-related data includes: minimum start timestamp, GC safe point, service safe point, which are stored in the respective etcd path of each keyspace in PD.

## Motivation

In order to facilitate multi-tenant deployment of TiDB in a shared TiKV cluster, TiDB now supports the keyspace feature. Currently, the TiDB cluster with keyspace can only utilize the GC calculated by the global GC worker. A global TiDB GC worker (a TiDB server without keyspace configuration) shared by all TiDB instances is responsible for calculating the global GC safe point and resolving locks, while each keyspace's TiDB instance has its own GC worker. The GC worker of each keyspace performs only one operation: utilizing the global GC safe point to execute "delete-range" operations within its keyspace ranges.

However, in this implementation, the calculation of the global GC safe point depends on the oldest timestamp (minStartTS, service safe point, etc.) used to calculate the GC safepoint of all TiDB clusters (including those with keyspace configured and those without keyspace configured). When any factor affecting the GC safepoint is blocked, the GC of all TiDB clusters will be blocked.

Therefore, we propose the **Keyspace Level GC**:

TiDB Side:
Isolate the GC safe point calculations and the storage of related data between keyspaces (the concept is `keyspace level GC`). This will not affect the GC safe point calculations of other keyspaces. Keyspaces can be created by setting `gc_management_type = keyspace_level_gc` to enable keyspace level GC. Then, this keyspace can calculate its own GC safe point.

TiKV Side:
In the GC process, it parses the keyspace ID from the data key, uses the keyspace meta config and the keyspace-level GC safe point corresponding to the keyspace ID to determine the GC safe point value of the data key and execute the GC logic.

## Implementation in TiKV

1. Get [keyspace metadata](https://github.com/pingcap/kvproto/blob/d9297553c9009f569eaf4350f68a908f7811ee55/proto/keyspacepb.proto#L26C1-L33C2) and keyspace-level GC safe points from PD:
   - Create a keyspace metadata watch service: Watch the keyspace metadata etcd path to get the keyspace metadata and store it in the mapping from keyspace ID to keyspace metadata.
   - Create a keyspace level GC watch service: Watch the corresponding etcd path to get the GC safe point for the keyspace and store it in the mapping from keyspace ID to keyspace level GC safe point.

2. How to get the GC safe point after getting the MVCC key in the Compaction Filter:
 ![img.png](../media/keyspace-level-gc-get-gc-safe-point.png)

3. How to determine if a keyspace uses a keyspace level GC safe point:
 ![img.png](../media/keyspace-level-gc-is-enable-keyspace-level-gc.png)

4. Use GC safe point to optimize trigger timing and assert judgment:
   1. In the global GC, it will skip GC when GC safe point is `0` in `create_compaction_filter`.
      After supporting keyspace level GC, GC is skipped if global GC safe point is 0 or if no keyspace level GC is initialized.
   2. In the global GC, `check_need_gc` will skip GC in the compaction filter when `props.min_ts` is greater than all GC safe points.
      After implementing keyspace level GC, if `props.min_ts` is greater than the global GC safe point and `props.min_ts` is greater than all keyspace level GC safe points, it will return false.
   3. In the global GC, assert( safe_point > 0 ) when new compaction filter.
      After supporting keyspace level GC, the assert condition is whether global GC safe point > 0 or any keyspace level GC safe point has been initialized.

5. Support the use of keyspace level GC for data import and export:
   - When using BR, CDC, Lightning, or Dumpling to import or export keyspace data, you need to update the service safe point for the specified keyspace. When the task starts, it needs to get the keyspace metadata first to determine whether global or keyspace GC is being used, and then execute the relevant GC logic on the corresponding GC safe point and service safe point.

## Global Backup

Specify ts to backup data for non-keyspaces and all keyspaces in the entire cluster. It will be introduced in another RFC.

[1]: https://github.com/tikv/rfcs/blob/master/text/0069-api-v2.md#new-key-value-codec
