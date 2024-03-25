# Apply Raft Log Before Persistence

## Summary

Allow committed raftlog to be applied before it is persisted to local storage.

## Motivation

In the cloud environment, we found a lot a performance issues related to disk IO latency jitters. And due to TiDB's distribution transaction mode, one transaction is likely to involve several regions which are likely distributed in several TiKV instances. Thus, one slow TiKV instance can severely implact the cluster's performance.

In the current implementation, TiKV can only apply a raft log after it is `committed` and `persisted`. Since `committed` already means that the log is persisted on quorum of instances, there is no need to wait the persistence of the current instance. Thus apply committed raft logs before they are persisted is a effective way to alleviate the performance jitter cause by random slow IO.

## Detailed design

The change of the raft itself is quite simple: only change the upper bound of available raft log from `min(committed, persisted)` to `committed`. While in theory we can do this for all raft log in any circumstances, we restrict the applicable conditions to make the implementation simpler and reduce unexpected corner cases. The applicable conditions are:

- Region leader. In most cases, only region leader handles foreground traffic, so do it on leader is enough. We maybe remove this restriction in the future so features such as `Follower Read` and `TiFlash` can benefit from it.
- Raft term should be the same. The applied raft log's term should be the same as the latest persisted raft term. 
- Limited gap between applied index and committed index. We expose a configuration to set the maxinum gap between applied index and persisted index. Too large gap many cause issue for region cache and raft log gc.

Besides, there are some corner cases that are hard to handle when raft log is lost(after restarting), so we may want to ensure the raft log is persisted before applying it:

- `PrepareMerge`. The `PrepareMerge` message itself is not the problem. But it makes it harder to handle `online unsafe recovery`, so we always apply `PrepareMerge` after the log is persisted. 
- `CompactLog`. raft log compaction is handled differently in leader and follower/learner:
  - Leader. When check and propose `CompactLog`, we ensure that compacted_index <= persisted_index.
  - Follower/Learner. Apply `CompactLog` after it is persisted. This is always true since we do not apply unpersisted raft log on follower. 

### Implementation Details

#### raft-rs

- Add a new configuration `apply_unpersisted_log_limit` and a new public function `set_apply_unpersisted_log_limit` to control how many committed raft logs that can be apply. The default values is 0 so the behavior stays the same as before.
- Change the upper bound of availabe raft log in `RaftLog::next_entries_since` and `RaftLog::has_next_entries_since` from `min(committed, persisted)` to `min(committed, persisted+apply_unpersisted_log_limit)`.
- Looses some checks at initializing since we can no longer keep the invariant that `applied_index <= committed_index` and `applied_index <= persisted_index`.

#### tikv

- Add a raftstore configuration `apply_unpersisted_log_limit` and add a variable `min_safe_index_for_unpersisted_apply` in raft fsm to track the target applied index. In following events, we may change `min_safe_index_for_unpersisted_apply` and raft's `apply_unpersisted_log_limit`(by calling `set_apply_unpersisted_log_limit`) in following scenarios:
  - Raft FSM initialization. set raft's `apply_unpersisted_log_limit` to 0 and `min_safe_index_for_unpersisted_apply` to current `last_index`.
  - `PerpareMerge`. After proposing `PerpareMerge`, set raft's `apply_unpersisted_log_limit` to 0 and set `min_safe_index_for_unpersisted_apply` to the message's index.
  - `on_leader_changed`. When raft leader changes, set raft's `apply_unpersisted_log_limit` to 0 and set `min_safe_index_for_unpersisted_apply` to current `last_index`.
  - `ApplyRes`. After handling `ApplyRes`, if `applied_index >= min_safe_index_for_unpersisted_apply`, set raft's `apply_unpersisted_log_limit` to the configed value.
- When check compact raft log, always ensure the proposed `compact_index` <= `persisted_index`. 
- Default configurations change.
 - `raftstore.cmd-batch-concurrent-ready-max-count` controls the number of ongoing raft batch while handling a new raft batch and the default value is 1. This default value is too small for the new feature so we will change it to 32 when `Async Raft IO` is enabled(by setting `raftstore.store-io-pool-size > 0`) and `apply_unpersisted_log_limit > 0`.


#### Feature Compatibility

##### Raft Entry Cache

Currently we only evict raft log in entry cache after it is persisted. When `apply_index` is ahead of `persist_index`, we may not be able to evict raft entries in the entry cahce. Thus, it is possible the memory used in entry cache is more than expected. 

By setting a proper `apply_unpersisted_log_limit`, we expected the entry cache won't consume to much memory. We may still need to enhance the memory management in the future if this can be a problem.

##### Compact Raft Log

After this change, if region leader's raft persistence is too slow and the gap between `applied_index` and `persisted_index` is too large these committed raft logs can be GCed as they are not persisted. So it is possible that the raft log(on other follower instances) may consume to many disk spaces. 

Again, we depends on a proper `apply_unpersisted_log_limit` to handle this issue currently. 

##### Online Unsafe Recovery

In the `online unsafe recovery` scenario when raft majority are lost, we try to recover the state machine from existing peers. In the current implementaion, we choose the peer whose `last_index` is the largest to force become leader and use its data to recover the missing peers. 

But after this change, the peer with the miximum `last_index` may not contains the most data if there is another peer whose `applied_index` is larger than the maxinum `last_index`. So, in order to recovery the most data, we add an extra rule when chooses leader:

  - if max(applied_index) > max(last_index), then choose the peer with the maximum applied_index.

In the recovery process, before force leader, if the current peer's `applied_index` > `last_index`(in this case the `committe_index` should be equal to `last_index`), which means that these raft logs are missing and is impossible to be recovered. To fix the raft state, we should fist advance the `committed_index` and `last_index` to `applied_index` so it can win an election. To fix the missing raft logs, we should also advance the `truncated_index` to `applied_index`, so other peer can sync to the same state by snapshot.

So we add following step to handle online unsafe recovery if `applied_index` > `last_index`:

- At `PreForceLeader` state, if `applied_index` > `last_index`, send a `ForceCompact` message to ApplyFsm and change the state to `ForceCompact`.
- When the ApplyFsm receive a `ForceCompact` message, it change the `committed_index` and `truncated_index` to the `applied_index` in ApplyState.
- When the peer receive the `ForceCompact`'s response from ApplyFsm, if the current state is `ForceCompact`, it sets its `last_index`,`commited_index` to `applied_index`, trucates raft log to `applied_index` and then enter `ForceLeader`.

## Drawbacks

In order to support this change, we need to loose some checks between raft's `applied_index`, `committed_index` and `persisted_index`. Thus, we may not able to find out some scenarios that the raft data is corrupted that lead to inconsistent data. 

## Alternatives

[double write file system](https://github.com/tikv/raft-engine/pull/323) tries to handle the IO problem by using two disks to store the raft log. The raft log is persisted if at least one of the disks finish persisting. This approach is more complex and is not commonly feasible as it needs an extra disk.

## Unresolved questions

- The extra memory useage by raft entry cache.
- The extra disk space by raft log compaction.
