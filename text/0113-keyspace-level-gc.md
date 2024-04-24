# Keyspace level GC

## Summary

- RFC PR:
- Tracking Issue: 

TiKV support keyspace level gc.

## Motivation

Previously, TiDB has supported the deployment of multiple TiDB clusters with different keyspaces in the TiDB configuration on a single PD TiKV cluster.

We've implemented multiple TiDB clusters, with one overall TiDB GC worker (TiDB without Keyspace configuration) pushing the global GC SafePoint, resolve locks, While each keyspace's TiDB depends on the global GC SafePoint, the keyspace's own TiDB gc worker only does deleteRange logic.

TiDB PR https://github.com/pingcap/tidb/pull/51300 implements: Isolation of GC SafePoint computations between keyspaces (the concept in the code is keyspace level gc). When the Keyspace GC SafePoint advancement of Keyspace with keyspace level gc is slow, it can not affect the GC SafePoint advancement of other keyspaces and clean data in time. Each keyspace TiDB cluster can initially create the keyspace if the keyspace meta config is set: gc_management_type = keyspace_level_gc, that is, the GC SafePoint that can advance its own Keyspace in the Keyspace dimension.

TiKV PR https://github.com/tikv/tikv/pull/16808 implemented on TiKV side: When GC keyspace data, It parses the keyspace id from the key, combines the keyspace meta cache and the keyspace level GC safepoint cache corresponding to the keyspace id to determine whether to use this keyspace keyspace_level_gc safepoint to gc.


## Two concepts:
1. Global GC:
    - Represents the previous default GC logic; there is a TiDB calculate the global gc safepoint for the whole cluster.
    - The default GC management type for keyspace is Global GC,
2. Keyspace level GC:
    - Indicates that the keyspace will advance its own gc safe point.
    - It is possible and only possible to set gc_management_type = keyspace_level_gc when PD creates keyspaces
    - The keyspace which already set gc_management_type = keyspace_level_gc that uses Global GC is not supported
    - Keyspace GC-related data minstartts, gc safepoint,service safepoint have their own storage path with keyspace prefix in PD. Thus the GC of keysapce can be advanced by itself.


## Implementation in TiKV
1. Get keyspace meta and keyspace level gc safepoint:
    - New KeyspaceMetaWatchService : Watch keyspace meta etcd path to get the keyspace meta and put it in cache keyspace_id_meta_map: <u32, keyspacepb::KeyspaceMeta>
    - New KeyspaceLevelGCWatchService : Watch the etcd path of the keyspace gc safe point to get the GC safe point with keyspace level GC enabled, Put in cache keyspace_level_gc_map: <u32, u64>,

2. How to get GC safepoint by mvcc key in Compaction Filter:
![img.png](../media/keyspace-level-gc-get-gc-sp.png)

3. How to determine if a keyspace uses a global gc safe point:
![img.png](../media/keyspace-level-gc-is-global-gc.png)