# Keyspace level GC

## Summary

- RFC PR: https://github.com/tikv/rfcs/pull/113
- Tracking Issue: https://github.com/tikv/tikv/issues/16896

TiKV support keyspace level GC.

## Concepts of GC management type:

1. Global GC:
   - Represents the previous default GC logic; there is a TiDB calculate the global GC safe point for the whole cluster.
   - The default GC management type for keyspace is Global GC.
2. Keyspace level GC:
   - Indicates that the keyspace will advance its own GC safe point.
   - Keyspace GC-related data: min start ts, GC safe point, service safe point, stored in own etcd path of each keyspace in PD.

## Usage and Compatibility
1. The GC management type of the keyspace is set in the config of the keyspace meta. It can be set with the key "gc_management_type". 
2. If you want to set 'gc_management_type' to 'keyspace_level_gc' for a keyspace, it can only be set when the keyspace is created. The GC management type of the keyspace cannot be changed after the keyspace has been created.

## Motivation

Previously, TiDB has supported the deployment of multiple TiDB clusters with different keyspaces 
on a single PD TiKV cluster.

In the whole of TiDB clusters, one global TiDB GC worker (A TiDB server without Keyspace configuration) is in charge of calculating the global GC safe point and resolving locks, While each keyspace's TiDB has their own GC Worker,
The GC Worker of each keyspace use the global GC safe point to do "delete-range" in its keyspace ranges.

But in this implementation the calculation of the global GC depends on the oldest safe point and min start ts of all keyspaces. When the GC safe points of any keyspace is slow, GC of all other keyspaces will be blocked.

So we propose the **Keyspace Level GC**:

TiDB side:
Isolate of GC safe point calculations between keyspaces (the concept is 'keyspace level GC').
it will not affect other keyspaces GC safe point calculation.
Keyspaces can be created by setting gc_management_type = keyspace_level_gc to enable keyspace level GC, then this keyspace can calculate GC safe point by itself.

TiKV side:
In GC process, it parses the keyspace id from the data key, combines the keyspace meta config and the keyspace level GC safe point corresponding to the keyspace id to determine the GC safe point value of the data key and execute the GC logic.

## Implementation in TiKV

1. Get keyspace meta and keyspace level GC safe point:
    - New KeyspaceMetaWatchService : Watch keyspace meta etcd path to get the keyspace meta and put it 
      in cache keyspace_id_meta_map<u32, keyspacepb::KeyspaceMeta>.
    - New KeyspaceLevelGCWatchService : Watch the etcd path of the keyspace GC safe point to get the GC safe point 
      with keyspace level GC enabled, put it in cache keyspace_level_gc_map<u32, u64>.

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

5. Other non-GC logic that uses GC safe point does not currently have to support keyspace level GC.
   1. check region consistency command: 
      1. It needs check GC safe point on followers to ensure that the data to be checked on the follower is not GC. 
      2. It doesn't support which keyspace with keyspace level GC enabled yet, if user request check consistency 
      for the keyspace range, the user will get a "not supported" message.
   2. GC safe point used in raftstore to trigger compaction when no valid split key can be found. 
      It was introduced in PR https://github.com/tikv/tikv/pull/15284
      1. It just uses GC safe point to determine when to trigger compaction. 
         The main PR of the Keyspace level GC will not fit this logic. 
         It will be considered for submitting another PR for support after the keyspace level GC core PR is merged.