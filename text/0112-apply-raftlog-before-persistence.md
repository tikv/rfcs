# Apply Raft Log Before Persistence

## Summary

- RFC PR: https://github.com/tikv/rfcs/pull/112
- Tracking Issue: https://github.com/tikv/tikv/issues/16717

Allow committed raftlog to be applied before it is persisted to local storage.

## Motivation

In the cloud environment, we have encountered performance issues with disk IO latency jitters in the cloud environment. Due to the distribution transaction mode of TiDB, a single transaction can involve multiple regions distributed across different TiKV instances. Therefore, if one TiKV instance is slow, it can significantly impact the overall performance of the cluster. Since we cannot control the IO latency and it is unlikely to be resolved in the near future, we should address this issue at the software level to mitigate its impact.

Currently, in the implementation of TiKV, a raft log can only be applied after it is both "committed" and "persisted". While "committed" already indicates that the log has been persisted on a quorum of instances, waiting for the persistence of the current instance is unnecessary. Therefore, applying committed raft logs before they are persisted is an effective approach to reduce performance jitter caused by random slow IO.

## Detailed design

The change of the raft itself is quite simple: only increase the upper bound of available raft log from `min(committed, persisted)` to `committed`. While in theory we can do this for all raft log in any circumstances, we restrict the applicable conditions to make the implementation simpler and reduce unexpected corner cases. The applicable conditions are:

- Region leader. In most cases, only region leader handles foreground traffic, so do it on leader is enough. We maybe remove this restriction in the future so features such as `Follower Read` and `TiFlash` can benefit from it.
- Raft term should be the same. The applied raft log's term should be the same as the latest persisted raft term. 
- Limited gap between applied index and committed index. We expose a configuration to set the maxinum gap between applied index and persisted index. Too large gap many cause issue for region cache and raft log gc.

Besides, there are some corner cases that are not easy to handle when raft log is lost(after restarting), so we may want to ensure the raft log is persisted before applying it:

- `PrepareMerge`. The `PrepareMerge` message itself is not the problem. But it makes it harder to handle `online unsafe recovery`, so we always apply `PrepareMerge` after the log is persisted. 
- `CompactLog`. raft log compaction is handled differently in leader and follower/learner:
  - Leader. When check and propose `CompactLog`, we ensure that compacted_index <= persisted_index.
  - Follower/Learner. Apply `CompactLog` after it is persisted. This is always true since we do not apply unpersisted raft log on follower. 

### Correctness

The order of local persistence and apply does not impact the raft replication protocol itself because the log is already committed(persisted on quorum instances). When an instance is not restarted, all unpersisted logs reside in memory, so it does not affect any actions that require reading the raft log. The only exception is the raft log GC, where persisted raft logs are truncated only on the leader and unpersisted logs are not applied on followers/learners.

In the case of restart, if applied raft log are not persisted, the initialized last index will fall behind the applied index, while the raft group's maxinum last index must be bigger than the applied index. So this peer is impossible to become the leader. Therefore, the peer can either catch up to the latest raft state by either replicating the missing raft logs or doing snapshot if any missing raft logs are already truncated.

Regarding administrative operations, special attention needs to be given to region merging because it involves two raft groups and uses a two-phase commit mechanism. Between the "PrepareMerge" and "CommitMerge" stages, there is an intermediate state called "CatchupLogs" that requires reading the delta raft logs and sending them to the target region leader. Thus, after a restart, the source peer may have lost some unpersisted logs, and the target region leader can be on the same instance. We can easily handle this by letting the source peer(as a follower) first catch up all missing logs from the leader and then proceed with the remaining steps. However, in "online unsafe recovery" scenarios, the situation may become more complex, and we will discuss this in the corresponding section.

### Implementation Details

#### raft-rs

- Add a new configuration `apply_unpersisted_log_limit` and a new public function `set_apply_unpersisted_log_limit` to control how many committed raft logs that can be apply. The default values is 0 so the behavior stays the same as before.
- Change the upper bound of availabe raft log in `RaftLog::next_entries_since` and `RaftLog::has_next_entries_since` from `min(committed, persisted)` to `min(committed, persisted+apply_unpersisted_log_limit)`.
- Looses some checks at initializing since we can no longer keep the invariant that `applied_index <= committed_index` and `applied_index <= persisted_index`.

#### tikv

- Add a raftstore configuration `apply_unpersisted_log_limit` and add a variable `min_safe_index_for_unpersisted_apply` in raft FSM to track the target applied index. For the following events, we may change `min_safe_index_for_unpersisted_apply` and raft's `apply_unpersisted_log_limit`(by calling `set_apply_unpersisted_log_limit`) in following scenarios:
  - Raft FSM initialization. set raft's `apply_unpersisted_log_limit` to 0 and `min_safe_index_for_unpersisted_apply` to current `last_index`.
  - `PerpareMerge`. After proposing `PerpareMerge`, set raft's `apply_unpersisted_log_limit` to 0 and set `min_safe_index_for_unpersisted_apply` to the message's index.
  - `on_leader_changed`. When raft leader changes, set raft's `apply_unpersisted_log_limit` to 0 and set `min_safe_index_for_unpersisted_apply` to current `last_index`.
  - `ApplyRes`. After handling `ApplyRes`, if `applied_index >= min_safe_index_for_unpersisted_apply`, set raft's `apply_unpersisted_log_limit` to the configed value.
- When check compact raft log, always ensure the proposed `compact_index` <= `persisted_index`. 
- Default configurations change.
 - `raftstore.cmd-batch-concurrent-ready-max-count` controls the number of ongoing raft batch while handling a new raft batch and the default value is 1. This default value is too small for the new feature so we will change it to 32 when `Async Raft IO` is enabled(by setting `raftstore.store-io-pool-size > 0`) and `apply_unpersisted_log_limit > 0`.


#### Feature Compatibility

##### Raft Entry Cache

Currently we only evict raft logs in entry cache that are already persisted. When `apply_index` is ahead of `persist_index`, we may not be able to evict raft entries in the entry cahce. Thus, it is possible that the memory usage is more than expected. 

By setting a proper `apply_unpersisted_log_limit`, we expected the entry cache won't consume too much memory. We may still need to enhance the memory management in the future if this becomes a problem.

##### Compact Raft Log

After this change, if the raft persistence of the region leader is excessively slow and there is a significant gap between the `applied_index` and `persisted_index`, these committed raft logs cannot be GCed as they are not persisted. So it is possible that the raft log(on other follower instances) may consume to many disk spaces.

Again, we depends on a proper `apply_unpersisted_log_limit` to handle this issue currently. 

##### Online Unsafe Recovery

Online unsafe recovery is a feature to recover the raft state machine from remaining peers when raft majority are down.

In the current implementaion, we choose the peer whose `last_index` is the largest to force become leader and use its data to recover the missing peers. 

But after this change, the peer with the miximum `last_index` may not contains the most data if there is another peer whose `applied_index` is larger than the maxinum `last_index`. So, in order to recovery the most data, we add an extra rule when chooses leader:

  - if max(applied_index) > max(last_index), then choose the peer with the maximum applied_index.

In the recovery process, before force leader, if the current peer's `applied_index` > `last_index`(in this case the `committe_index` should be equal to `last_index`), which means that these raft logs are missing and is impossible to be recovered. To fix the raft state, we should fist advance the `committed_index` and `last_index` to `applied_index` so it can win an election. To fix the missing raft logs, we should also advance the `truncated_index` to `applied_index`, so other peer can be synced to the same state by snapshot.

So we add following step to handle online unsafe recovery if `applied_index` > `last_index`:

- At `PreForceLeader` state, if `applied_index` > `last_index`, send a `ForceCompact` message to ApplyFsm and change the state to `ForceCompact`.
- When the ApplyFsm receive a `ForceCompact` message, it force change the `committed_index` and `truncated_index` to the `applied_index` in ApplyState.
- When the peer receive the `ForceCompact`'s response from ApplyFsm, if the current state is `ForceCompact`, it sets its `last_index`,`commited_index` to `applied_index`, trucates raft log to `applied_index` and then enter `ForceLeader`.

###### Region Merge

In the current implementaion, when doing unsafe recovery, we depend on the normal region merge procedure to either finish or rollback the merge. The only difference is that the remain active peers' count is not enough to form a quorum. We handle this by force advance the committed index iff all remain peers has persisted the raft log.

But in case that `PrepareMerge` is committed, it is possible that all the remain peers do not contain all the raft logs and their applied index can also be different, only some of the peers has applied the `PrepareMerge` log. Thus we need to both recover the region's state and resolve the merge state. We have two possible ways to handle this:

1) First sync all the remain source peers' state to `PrepareMerge`. This approach need to allow doing snapshot in `PrepareMrege` state and add a new mechanisum to let snapshot also change the raft state(to `PrepareMerge`). After finish syncing to the same state, we can just commit or rollback merge with the current mechanism.

2) First rollback the merge on the newest source peer and target region and then fix the region state since now the region are in normal state. In this approach, we need decice how to rollback all the source peers that have applied the `PrepareMerge`and how to schedule the `Rollback` merge to target region's leader as the source peer on that instance may not applied the `PrepareMerge`.

In both approach, we need to introduce new mechanisum to resolve the merge state, this is like to let the already complex mechanism complicated. While region merge is a rare operation, we think just fallback to always wait persistence before applying is a better choice.

## Drawbacks

In order to support this change, we need to loose some checks between raft's `applied_index`, `committed_index` and `persisted_index`. Thus, we may not able to find out some scenarios that the raft data is corrupted that lead to inconsistent data. 

## Alternatives

[double write file system](https://github.com/tikv/raft-engine/pull/323) tries to handle the IO problem by using two disks to store the raft log. The raft log is persisted if at least one of the disks finish persisting. This approach is more complex and is not commonly feasible as it needs an extra disk.

## Unresolved questions

- The extra memory useage by raft entry cache.
- The extra disk space by raft log compaction.
