# Follower/Learner request snapshot actively

## Summary

In current design, a follower can only wait for the leader to send a snapshot
iff the leader finds out the follower cannot catch up to its index. This RFC
proposes to let a follower be able to request snapshot from leader actively.

## Motivation

For the TiFlash project, which synchronizes data via learner's raft log, the
region meta data is stored in memory and flushed to disk periodically, chances
are that the region meta is lost during the process. In the other hand, we
also need a solution to restore broken data written to TiFlash since TiFlash
only owns one copy. Provided that the learner is able to request snapshot
actively, we can help restore broken data, and the restore process will be
faster and easier once TiFlash provides consistency check.

## Detailed design

### Request snapshot

We should add a new API for raft-rs to request a snapshot of a raft.

The request snapshot message is piggybacked by `reject` and `reject_hint`, it
sets `reject` to `true` and `reject_hint` to `INVALID_INDEX`.

Also, a new field `request_snapshot` is needed in the `eraftpb.Message`,
in order to distinguish between the reject messages from the new follower
(A new follower may response `reject_hint = INVALID_INDEX` too).

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

Should a follower be able to start a leader election? It's up to callers to
decide. E.g., a follower can always request a snapshot even if it's
totally fine. However, if a follower loses it's raft logs, it should not start
a leader election.

### Snapshot

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

## Unresolved questions

The cost of requesting and applying snapshot. (when multiple region snapshots
are requesting or multiple snapshots of same region is requesting)