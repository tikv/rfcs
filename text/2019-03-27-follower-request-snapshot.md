# Follower/Learner request snapshot actively

## Summary

In current design, a follower can only wait for the leader to send a snapshot
iff the leader founds out the follower cannot catch up to its index. This RFC
proposes to let a follower able to request snapshot from leader actively.

## Motivation

For the TiFlash project, which synchronizes data via learner's raft log, the
region meta data is stored in memory and flushed to disk periodically, chances
are that the region meta is lost during the process. In the other hand, we
also need a solution to restore broken data written to TiFlash since TiFlash
only owns one copy. Provided that the learner is able to request snapshot
actively, we can help restore broken data, and the restore process will be
faster and easier once TiFlash provides consistency check.

## Detailed design

### raft-rs

We should add a new API for raft-rs to request a snapshot of a region via grpc.

The request snapshot message is piggybacked by `reject` and `reject_hint`, it
sets `reject` to `true` and `reject_hint` to `INVALID_INDEX`.

When the follower requests a snapshot, it enters a special state called
`requesting_snapshot`. In this state, the follower rejects all `MsgAppend` and
responses a request snapshot message.

#### Flow control

Snapshot needs flow control, otherwise, it wastes disk and network bandwidths by
repeatedly sending snapshot and raft logs.

Luckily, raft-rs already provides an efficient flow control mechanism, request
snapshot is also managed by the mechanism. Request snapshot messages will
eventually let the follower enter a `Snapshot` state in a leader's view, thus
leader sends a snapshot and pauses replicating raft logs to the follower.

#### Leader election

A follow shall not start a leader election if it's in the`requesting_snapshot`
state, because its raft logs or state machine may be damaged. Transfer leader
messages will silently be ignored, unfortunately.

### TiKV

The actual raft snapshot is generated in TiKV side, it is important to not
generate a **stale** snapshot. By stale, it means the `snapshot_index` is less
than the current commit index. Otherwise, followers may apply some already
applied admin commands, like split or merge. It is an undesired behavior to
apply admin commands repeatedly.

## Drawbacks

Enabling followers/learners to request snapshot actively could affect some
pre-assumptions for TiKV leaders potentially. The TiKV might, for instance,
lost control of the number of generated snasphots, or affect other requests
when too many snapshot requests are processing. Further discussions should aim
at these topics to resolve corner cases above.

## Alternatives

### Plan B

In order to request for a region's snapshot from leader, TiKV learner may also
deceive the leader by replying an index much previous than the leader's index
so that the leader will send a snapshot.

The pros and cons of Plan B is that it is easy to implement but leaves
uncertainty because the leader may not really respond by sending a snapshot.

## Unresolved questions

The cost of requesting and applying snapshot. (when multiple region snapshots
are requesting or multiple snapshots of same region is requesting)