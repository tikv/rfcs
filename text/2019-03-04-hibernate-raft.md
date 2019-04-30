# Hibernate Region

## Summary

This RFC proposes to hibernate a peer when it reaches a stable state.

## Motivation

There are several ticks when driving a peer. The cost of ticks is usually
small. However, when there are a lot of regions in one TiKV instances, the
cost can be significant. We have observed that an idle cluster with many
regions can waste a lot of CPU time to send useless heartbeats and tick for
routine checks. In an online cluster with tens of thousands of regions,
enlarging the ticks' interval can usually lead to better performance. Threaded
raftstore can ease the problem but can't solve it. In general, not all regions
keep working all the time. Unbalanced load is a very common use case in
practice. For those idle regions, we can hibernate them to avoid wasting
resources and get better performance.

## Detailed design

A peer is driven by several ticks. Including Raft, RaftLogGc,
SplitRegionCheck, PdHeartbeat, CheckMerge, CheckPeerStaleState.
Following describes how to disable most of these ticks when possible.

### Raft

Raft tick is used to drive a raft state machine, so a leader can send
heartbeats, and a follower can campaign when the leader is down. To make sure
raft can still work properly, we can only disable ticks when the group is in a
steady state. From a peer's perspective, a group's state can be defined as Ordered,
Chaos and Idle. Ordered means the cluster is working in normal fashion, and everyone
is ticking actively. Chaos means leadership may not be hold, and a new leader may have
been elected. Idle means there is no writes for a long time that tick is disabled. We
use the phrase like "become/is Chaos/Ordered/Idle" to express "considers the group is
in Chaos/Ordered/Idle state".

Internally, raft follower will reset elapsed election ticks when receives messages from
leader. So for followers, it becomes Ordered when it admits a leader and receives
messages from leader. For leader, it should always be Ordered until explicitly
transiting to other state.

If a follower receives a proposal or vote messages, it means leader is not working, so
it should become Chaos. Specially, all nodes should be Chaos when started. If leader
receives a vote message, it means some follower is trying to steel the leadership,
so the group state should be Chaos too.

When the group is in Ordered state, and there is no further writes, the group can become
Idle state. But leader needs to replicate logs to followers, so it should not be Idle
until all the active followers have received latest logs and it applies to latest logs.
Note that only active followers need to be considered as inactive followers may be
isolated, keep ticking may not make a difference most of time. Apply to latest logs can
ensure all configuration is applied.

Because leader only considers active followers, so when inactive followers rejoin the
group, leader may not replicate logs to the follower in time. If the follower is Chaos,
it can use vote message to make leader Chaos too, and start replication again. If the
follower is Idle it may not receive any logs until there is write from client again.
So leader needs to find a way to see if a follower is active again. So even leader
is in Idle state, it should broadcast heartbeats to all followers in a long period.

Followers can't know whether there is further write, so it can just consider the group
Idle when it contains logs from current term and has a leader. And then let leader to
drive the log replication. And even followers are in Idle state, it should also check
whether heartbeats arrives in long enough time. If not, it means the follower may have
been removed from the group or leader is down. It should become Chaos and campaign.

### RaftLogGc

Because raft log is done by proposal, so only leader needs to tick. And when
raft tick is stopped, this tick can be stopped too. When new write proposal
arrives or a new leader is elected, the tick can be started again.

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

This tick can't be removed as there is no reliable way to remove stale peer.

Besides leader has to broadcast heartbeats to all followers in a long period
as memtioned above, so this tick can be also used for broadcasting and checking
heartbeats. To make sure followers check the leader heartbeat reliablely, we can
add a intermediate state to group state as PreChaos. If a follower is Idle, then
change it to PreChaos; if it's PreChaos, then change it to Chaos. So it takes
two ticks for follower to actually start campaign while it only takes one tick
for leader to broadcast heartbeats. So followers should generally stay calm if
network and clock is good.

## Drawbacks

When a leader is down, it may take a longer time to recover than before.

## Alternatives

### Merging small packets

It's expected to reduce gRPC thread usages a lot. However, it does not help to
reduce raftstore's CPU usages. Though it may still be a good solution to solve
the CPU consumption when all regions are under high load.

## Unresolved questions

When a node is cold restarted, there may still be a lot of CPU usages.
