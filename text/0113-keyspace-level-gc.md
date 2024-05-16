# Keyspace level GC

- RFC PR: https://github.com/tikv/rfcs/pull/113
- Tracking Issue: https://github.com/tikv/tikv/issues/16896

## Summary

TiKV support [keyspace][1] level GC (garbage collection).

## Concepts of GC management type

1. Global GC:
   - Represents the previous default GC logic; there is a TiDB calculate the global GC safe point for the whole cluster.
   - The default GC management type for keyspace is Global GC.
2. Keyspace level GC:
   - Indicates that the keyspace will advance its own GC safe point.
   - Keyspace GC-related data: min start ts, GC safe point, service safe point, stored in own etcd path of each keyspace in PD.

## Motivation

In order to support multi-tenat deployment of TiDB in a shared TiKV cluster, TiDB now supports keyspace feature. A global TiDB GC worker (A TiDB server without Keyspace configuration) shared by all TiDB is in charge of calculating the global GC safe point and resolving locks, While each keyspace's TiDB has their own GC Worker. The GC Worker of each keyspace use the global GC safe point to do "delete-range" in its keyspace ranges.

$$
GCSafePoint = Min\left\{GlobalSafePoint,\forall GCSafePoint\in Keyspaces\right\}
$$

But in this implementation the calculation of the global GC depends on the oldest safe point and min start ts of all keyspaces. When the GC safe points of any keyspace is small, GC of all other keyspaces will be blocked.

So we propose the **Keyspace Level GC**:

TiDB side:
Isolate of GC safe point calculations between keyspaces (the concept is 'keyspace level GC').
It will not affect other keyspaces GC safe point calculation.
Keyspaces can be created by setting gc_management_type = keyspace_level_gc to enable keyspace level GC, then this keyspace can calculate GC safe point by itself.

TiKV side:
In GC process, it parses the keyspace id from the data key, use the keyspace meta config and the keyspace level GC safe point corresponding to the keyspace id to determine the GC safe point value of the data key and execute the GC logic.

## Upgrade from `global GC` to `keyspace level GC`

1. Firstly, the update of global GC safe points and service safe points in PD should be stopped. The global GC/Service safe point can only be read. The global GC safe point is recorded as t1.
2. Stop the BR, CDC, Lightning, Dumpling tasks for the specified keyspace which need to enable keysapce level GC.
3. Use t1 to update the keyspace level GC safe point and GCWorker service safe point in the etcd with the specified keyspace.
4. Update 'gc_management_type' = 'keyspace_level_gc' of the specified keyspace meta config.
5. TiKV and TiFlash use keyspace level GC safe point perform GC.
6. BR, CDC, Lightning, Dumpling tasks for the specified keyspace can be started.
7. PD can restart to update global GC/Service safe point.

## Implementation in TiKV

1. Get keyspace meta and keyspace level GC safe point:
    - Create a keyspace meta watch service: Watch keyspace meta etcd path to get the keyspace meta and put it in the mapping from keyspace id to keyspace meta.
    - Create a keyspace level gc watch service : Watch the etcd path of the keyspace GC safe point to get the GC safe point with keyspace level GC enabled, put it in the mapping from keyspace id to keyspace level gc.

2. How to get GC safe point by mvcc key in Compaction Filter:
 ![img.png](../media/keyspace-level-gc-get-gc-sp.png)

3. How to determine if a keyspace uses a global GC safe point:
 ![img.png](../media/keyspace-level-gc-is-global-gc.png)

4. Use GC safe point to optimize trigger timing and assert judgment:
   1. In the global GC, it will skip GC when GC safe point is 0 in create_compaction_filter.
      After supporting keyspace level GC, GC is skipped if global GC safe point is 0 or if no keyspace level GC is initialized.
   2. In the global GC, check_need_gc function return false in create_compaction_filter.
      After supporting keyspace level GC, if props.min_ts > global GC safe point and props.min_ts > all keyspace level GC safe point will return false.
   3. In the global GC, assert( safe_point > 0 ) when new compaction filter.
      After supporting keyspace level GC, the assert condition is whether global GC safe point > 0 or any keyspace level GC safe point has been initialized.

5. Support use keyspace level GC for data import and export:
   1. When using BR, CDC, Lightning, Dumpling to import and export data specified keyspace, you need to update the service safe point for the specified keyspace. When the task starts, When the task starts, it needs to get keyspace meta first to determine whether to execute the keyspace level gc logic.
   2. For global BR, the global BR service safe point will also be required as one of the conditions of the block keyspace level GC, when the keyspace is required to update the service safe point.

[1]: https://github.com/tikv/rfcs/blob/master/text/0069-api-v2.md#new-key-value-codec
