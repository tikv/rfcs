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

```protobuf
message getSnapshotRequest {
    metapb.Region region = 1
}
```

### tikv learner

`tikv-rngine` is the specific learner that TiFlash works along with. TiFlash
will send a request to learner and ask for snapshot, and `rngine` shall request
this snapshot from the leader/followers.

Hereby we try to make things easier by only implementing requesting snapshot
from leader at first, further logic for requesting snapshot from other
followers can be discussed in the future.

### tikv master

TiKV leaders should answer the `getSnapshotRequest` by responding a
`SnapshotResponse` message and the follower/learner should do `ApplySnapshot`
logic as usual. After the process, Old snapshots may be GCed.

### error handling

Common errors that may encounter snapshot might fail to generate
(AsyncSnapshotErr), etc. Most logic does not need changing when we deal with
`getSnapshotRequest`. TiFlash needs to deal with extra errors when unable to
get and apply snapshot data.

## Drawbacks

Enabling followers/learners to request snapshot actively could affect some
pre-assumptions for TiKV leaders potentially. The TiKV might, for instance,
lost control of the number of external connections, or affect other requests
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