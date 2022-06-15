# Online Unsafe Recovery

- Tracking Issue: https://github.com/tikv/tikv/issues/10483

## Summary

Make TiKV cluster be capable of recovering from Raft groups from majority failure online

## Motivation

Currently, we have a tool to recover Raft groups from majority failure:
```
$ tikv-ctl unsafe-recover remove-fail-stores -s <store_id, store_id> …
$ tikv-ctl recreate-region -r <region_id> ...
```

The first command is used to recover Raft groups of which the majority but not all replicas failed. Its mechanism is removing failed stores from membership configurations of those Raft groups.

The second command is used to recover Raft groups of which all replicas failed. Before running the command we need to find those Raft groups, which is not a documented routine. So there are no users who can do this recovery without our DBAs’ help.

Both of the two commands should be run on stopped TiKV instances, which makes the recovery process so long and complex that not all users can perform all steps correctly.

And there are some corner cases that could lead to inconsistent membership configuration among all replicas. If the Region splits later, this mistake will lead to inconsistent Region boundaries, which is very hard to locate and fix.

So we need to re-design and re-implement a new feature to recover a TiKV cluster from majority failure. The feature should be:

- able to be executed online
- able to avoid all inconsistencies
- a feature instead of a tool, focus on Raft majority failure, shouldn’t be used for other situations
- easy to use, at most 1 or 2 steps or command

## Spec

We must know there are no silver bullets for Raft majority failure. There must be something lost after a TiKV cluster recovers from that. Considering such integrations or consistencies:

- Committed writes can be lost, so linear consistency of Raft can be broken
- Some prewrites can be lost for a given transaction, so Transaction Atomicity and/or Consistency can be broken
- Commits can be lost for a given transaction, so Transaction Consistency and/or Durability can be broken
- For a given table, both records or indices can be lost, so Data Consistency can be broken
- For a given transaction, it can be partially committed, so Data Integrity can be broken

## Interface

The entrance of this feature is on PD side, triggered by pd-ctl. There are two pd-ctl commands for it:
```
$ pd-ctl unsafe remove-failed-stores <store-id1, store-id2, store-id3...> --timeout=300 
$ pd-ctl unsafe remove-failed-stores show
```

The first command registers a task into PD. It's possible to call it again before the previous one gets finished, in which case PD just rejects the later tasks. `--timeout` specifies the maximum execution time for the recovery. If it doesn't finish the work within timeout, it fails directly.
The second command shows the progress the recovery, including all the operations have done and current operations being executing for the recovery.

## Detailed design

The changes fall into both PD and TiKV. A recovery controller is introduced in PD to drive the recovery process. Quite like other PD schedulers, it collects region information first, and dispatches corresponding operations to TiKV. Whereas regions losing leader won't trigger region heartbeat, the peers' information are collected by store heartbeat requests and the recovery operations are dispatched by the store heartbeat responses.

Moreover, given there is no leader for the regions losing majority peers, how can the rest of peers in one region execute the recovery operations consistently? To solve that, we introduce force leader. It will be elaborated in later section. Now all you need to know is that it forces Raft to make one peer to be leader even without the quorum. With force leader, the Raft group can continue drive the proposal to be committed and applied. So we can propose conf change to demote the failed voter to learner. After the conf change is applied, the quorum can be formed. Then let the force leader step down and it will elect a new leader by Raft normal process. So far, the recovery for one region is finished which is totally online.

Here is the overall process:

1. Pd-ctl registers a recovery task in recovery controller in PD
2. Recovery controller asks all the TiKVs by store heartbeat responses to send peers' information
3. Each TiKV collects all the peers on itself and sends the information called store report by store heartbeat response
4. PD receives the store report, and generates force leader recovery plan once receives all the store reports 
5. PD dispatch the force leader operations to TiKV by store heartbeat response
6. TiKV executes the force leader operations, and sends the latest store report 
7. PD receives the store report, and generates demote failed voters recovery plan once receives all the store reports 
8. TiKV executes the demote failed voters recovery operations, and sends the latest store report
9. PD receives the store report, and create empty region recovery plan once receives all the store reports 
10. TiKV executes the create empty region recovery operations, and sends the latest store report
11. PD receives the store report, and finish the recovery task

### Protocol

Add store report in `StoreHeartbeatRequest`

```protobuf
message StoreHeartbeatRequest {
    ...

    // Detailed store report that is only filled up on PD's demand for online unsafe recovery.
    StoreReport store_report = 3;
}

message PeerReport {
    raft_serverpb.RaftLocalState raft_state = 1;
    raft_serverpb.RegionLocalState region_state = 2;
    bool is_force_leader = 3;
    // The peer has proposed but uncommitted commit merge.
    bool has_commit_merge = 4;
}

message StoreReport {
    repeated PeerReport peer_reports = 1;
    uint64 step = 2;
}
```

Add recovery plan in `StoreHeartbeatResponse`

```protobuf
message StoreHeartbeatResponse {
    ...
    
    // Operations of recovery. After the plan is executed, TiKV should attach the 
    // store report in store heartbeat.
    RecoveryPlan recovery_plan = 5;
}

message DemoteFailedVoters {
    uint64 region_id = 1;
    repeated metapb.Peer failed_voters = 2;
}

message ForceLeader {
    // The store ids of the failed stores, TiKV uses it to decide if a peer is alive.
    repeated uint64 failed_stores = 1;
    // The region ids of the peer which is to be force leader.
    repeated uint64 enter_force_leaders = 2;
}

message RecoveryPlan {
    // Create empty regions to fill the key range hole.
    repeated metapb.Region creates = 1;
    // Update the meta of the regions, including peer lists, epoch and key range.
    repeated metapb.Region updates = 2 [deprecated=true];
    // Tombstone the peers on the store locally.
    repeated uint64 tombstones = 3;
    // Issue conf change that demote voters on failed stores to learners on the regions.
    repeated DemoteFailedVoters demotes = 4;
    // Make the peers to be force leaders.
    ForceLeader force_leader = 5;
    // Step is an increasing number to note the round of recovery,
    // It should be filled in the corresponding store report.
    uint64 step = 6;
}
```

### PD Side

PD recovery controller is organized in state machine manner, which has multiple stages. And the recovery plan is generated based on the current stage. Due to unfinished split and merge, or retry caused by network or TiKV crash, the stage may change back and forth multiple times. So the stages transition would be like this: 

```
  +-----------+
  |           |
  |   idle    |
  |           |
  +-----------+
        |
        |
        |
        v            +-----------+
  +-----------+      |           |          +-----------+           +-----------+
  |           |----->|   force   |--------->|           |           |           |
  |  collect  |      | LeaderFor |          |  force    |           |  failed   |
  |  Report   |      |CommitMerge|    +-----|  Leader   |-----+---->|           |
  |           |      |           |    |     |           |     |     +-----------+
  +-----------+      +-----------+    |     +-----------+     |
                          |           |        |     ^        |
                          |           |        |     |        |
                          |           |        |     |        |
                          |           |        v     |        |
                          |           |     +-----------+     |
                          |           |     |           |     |
                          |           |     |  demote   |     |
                          |           +-----|  Voter    |-----+
                          |           |     |           |     |
                          |           |     +-----------+     |
                          |           |        |     ^        |
                          |           |        |     |        |
                          |           |        v     |        |
                          |           |     +-----------+     |
  +-----------+           |           |     |           |     |
  |           |           |           |     |  create   |     |
  | finished  |           |           |     |  Region   |-----+
  |           |<----------+-----------+-----|           |
  +-----------+                             +-----------+
```

Stages are:
- idle: initiated stage, meaning no recovery task to do
- forceLeader: force leaders on the unhealthy regions whose the majority of peers are lost
- forceLeaderForCommitMerge: same as forceLeader stage, but it only force leaders on the unhealthy regions which has unfinished commit merge, to make sure the target region is forced leader ahead of the source region so that target region can catch up log for source region successfully.
- demoteVoter: propose conf change to demote the failed voters to learners for the regions having force leader
- createRegion: recreate empty regions to fill the range hole for the case that all peers of one region are lost.
- failed: recovery task aborts with an error
- finished: recovery task has been executed successfully

The recovery process starts from the collectReport stage. Apart from idle, finished and failed stages, each stage collects store reports first and then try to generate plan for different operation one by one in order that force leader comes first, then demote voter, and last create region. If there is a plan, it transfers into corresponding stage and dispatches the plan to TiKV. If there isn't any plan to do, the recovery process transits into failed or finished stage.

#### Pause conf-change, split and merge

Note that the healthy regions are still providing service, so there may be some splits, merges and conf-changes. These make the region range and epoch changes from time to time. As the store reports are collected from different stores at a different time point, they don't form a global snapshot view of any time point. To solve that, just pause all the scheduler and checker in PD and reject `AskBatchSplitRequest` to pause region split as well in process of recovery.

After the stage is turned into finished or failed, all schedulers and checker resume and don't reject `AskBatchSplitRequest` anymore.

### TiKV Side

When TiKV receives recovery plan from PD, it dispatches related operations to corresponding peer fsm and broadcasts message to all peers to send report after having applied to the latest index.

Here are the operations:
- WaitApply: to get the latest peer information, need wait until the apply index equals to commit index as least. 
- FillOutReport: get the region local state and raft local state of the peer
- Demote: Propose the conf-change that demotes the failed voters to learners, then the region is able to elect leader normally. If the region is already in joint consensus state, exit joint state first. After demotion, exit force leader state.
- Create: create a peer of new region on the store locally
- Tombstone: tombstone the local peer on store

There are three possible work flows:
- report phase(recovery plan is empty)
    1. WaitApply
    2. FillOutReport
- force leader phase(recovery plan only has force leader)
    1. EnterForceLeader
    2. WaitApply
    3. FillOutReport 
- plan execution phase(recovery plan has demotes/creates/tombstones)
    1. Demotes/Creates/Tombstones
    2. ExitForceLeader
    3. WaitApply
    4. FillOutReport

As you can see, no matter what, it always does the recovery operations(if any) and wait apply then fill out and send the store report. Here are more details about the force leader.

#### Force leader

The peer to be force leader is chosen by PD recovery controller following the same way as Raft algorithm does that the one has largest last_term or last_index.

Force leader is a memory state presented on the leader peer, it won't be replicated through Raft to other peers. In force leader state, it rejects write and read requests, and only accepts conf-change admin commands. 

The process of force leader is:
1. Wait some ticks until election timeout is triggered, to make sure the origin leader lease is expired
2. Pre force leader check that request vote to rest alive voters, and expect  all received are grant votes.
3. Be in force leader state, and forcibly forward commit index when logs are replicated to all the rest alive voters.

Note:
- The peer having latest up-to-date log may be a learner, learner could be a force leader.
- After being force leader, the commit index is advanced and some admin commands, e.g. split and merge, may be applied after that. For the newly split peer, it won't inherit the force leader state. If the newly split region can't select neither, it will be dispatched force leader operation in next stage of recovery process.
