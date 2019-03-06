# Hibernate Region

## Summary

This RFC proposes to hibernate a peer when it reaches a stable state.

## Motivation

There are several ticks when driving a peer. The cost of tick is usually
small. However, when there are a lot of regions in one TiKV instances, the
cost can't be eligible. We have observed that an idle cluster with many
regions can waste a lot of CPUs to send useless heartbeats and tick to routine
checks. Even in an online cluster with tens of thousands of regions, enlarge
the ticks' interval can usually lead to better performance. Threaded raftstore
can ease the problem but can't solve it. In general, not all regions keep
working all the time. Imbalanced load is a very common use case in practice.
For those idle regions, we can hibernate them to avoid resources waste, hence
get a better performance.

## Detailed design

A peer is driven by several ticks. Including Raft, RaftLogGc,
SplitRegionCheck, PdHeartbeat, CheckMerge, CheckPeerStaleState.
Following describes how to disable all these ticks when possible.

### Raft

Raft tick is used to driving a raft state machine, so leader can send
heartbeats, follower can campaign when leader is down. To disable it, we need
to make sure raft can still work properly. I suggest adding following rules to
make it work:

1. When a leader finds all followings are satisfied when handling ticks,
   it stops ticking.

    1) All followers have met commit_index == last_index_of_leader,
    2) apply_index == last_index,
    3) its flag named `waken` is false,

2. When a follower finds all followings are satisfied when handling ticks,
   it stops ticking.

    1) term == last_log_term,
    2) has a valid leader,
    3) its flag named `waken` is false,

3. When a peer is created, it starts ticking.

4. When a peer receives a proposal (write proposal/read index), it marks flag
   `waken` to true and starts ticking.

5. When a peer receives RequestVote/PreRequestVote/MsgTimeoutNow, it marks
   flag `waken` to true and starts ticking.

6. When a follower receives leader's message, it marks `waken` to false.

7. When a leader finds all nodes are actively replying messages, it marks
   `waken` to false.

When a cluster is started, because of 3, all peers are ticking. It works
just the old way. Because of 2, ticks of followers will not stop until a
leader is elected and at lease a new log is appended. If there is no further
proposals, because of 1, leader will stop ticking too.

When both leader and followers are hibernated, a proposal arrives. Leader will
start ticking again because of 4. And followers will tick. Note that leader
must tick in this case so that it can keep retry sending out entries when
network is in bad condition. And after the proposal is committed on all peers
and applied on leader, leader will stop ticking because of 1.

If during that time, leader is out of service, client will try to redirect
requests to followers. Because of 4, followers will start ticking. They start
campaigns in the end. And their campaign messages will wake the whole cluster
up because of 5. Note that in such case, new leader will not stop ticking even
it is elected and apply the new entry because there is still one out of service
node, which is the old leader. Rule 4 can be more aggressive like if it detects
that it has not received messages from leader for more than election_timeout,
it can start campaign immediately.

If during that time, follower is out of service and recovered before any
proposals, it will start ticking because of 3. To prevent it from disturbing
the leader, PreVote needs to be enabled. And leader will eventually be woken
up and broadcasting heartbeat until all followers have been calmed down and go
to sleep as 7 indicates. If follower is recovered after proposals, leader has
been woken up already, and the whole cluster will be made sure to be up to
date.

If during that time, a new node is added to the cluster, because of 1, leader
will keep ticking until the conf change is complete and new node has caught up
all the logs. If a node is removed, and the node is isolated, we need another
mechanism to make it removed. This will be discussed when talking about
`CheckPeerStaleState`.

### RaftLogGc

Apparently, only leader needs to tick. And when raft tick is stopped, this
tick can be stopped too. When new write proposal arrives or a new leader
is elected, the tick can be started again.

### SplitRegionCheck

This can be handled using the same policy as `RaftLogGc` except that when
rocksdb finishes compaction or merge/split is finished, it needs to tick again
at least one time.

### PdHeartbeat

This is also similar to `RaftLogGc`. However, `PdHeartbeat` must tick when
there are any read proposals to let pd rebalance correctly.

### CheckMerge

It has been implemented in the lazy way already that it's only scheduled when
a `PreMerge` is proposed while `CommitMerge` is not applied.

### CheckPeerStaleState

Hibernation means it should not be interrupted only when it's really
necessary. This tick is deprecated completely. We have two ways to clean up
stale peers without ticks.

One way is relying on the PD, so that when pd finds that the peer count of a
store doesn't match the expected value calculated from heartbeats of leader,
it sends a list of regions to store, so that store can clean all stale peers
according to the region epoch.

The other way is relying on the range check. If a store receives a demand to
create a new peer on the node while the range is covered by other peers, then
it asks the overlapped peer to check if it's stale. This is simple but may not
release space in time.

## Drawbacks

When a leader is down, it may take a longer time to recover than before.

## Alternatives

### Merging small packets

It's expected to reduce gRPC thread usages a lot. However, it does not help to
reduce raftstore's CPU usages. Though it may still be a good solution to solve
the CPU consumption when all regions are under high load.

## Unresolved questions

When a node is cold restarted, there may still be a lot of CPU usages.
