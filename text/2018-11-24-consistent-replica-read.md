# Consistent Replica Read

## Summary

*Consistent Replica Read* supports consistent read on the follower replica or
learner replica, which can reduce the read load on the leader. *Consistent
Replica Read* also useful in OLAP scenario or backup scenario.

The key to achieve consistent read on the replica is `ReadIndex`, which
introduced in chapter 6 of the [raft
thesis](https://ramcloud.stanford.edu/~ongaro/thesis.pdf).

## Motivation

Currently, all read requests are handled on the leader, and load balanced only
by the `balanceLeader` operator, there are lots of limitations. On one hand, we
want a way to improve read throughput and divert load away from the leader, on
the other hand we want some nodes to support read-only service and handle some
heavy analytics queries.

## Detail design

### Replica Read

As summary said, the key to implementing *Consistent Replica Read* is
`ReadIndex`, which already implemented in our
[raft-rs](http://github.com/pingcap/raft-rs) library. For now, some read
requests are already handled using the `ReadIndex` policy to handle, It follows
these steps:

1. Determine the policy of the read request. If the request is required to
   check the quorum, the replica is a leader and expired the lease, or the
   replica is a follower or learner, it should use `ReadIndex` policy.
2. Got the latest `CommitIndex` from the raft library. Assign a `uuid` for the
   request and invoke the `read_index` API of the raft library with the data
   thatwas encoded by `uuid`. then the raft state machine will notify our apply
   state machine with the `read_status` through the `Ready` message. and we can
   get the latest `CommitIndex` throught the corresponding `uuid` from the
   `read_status`.
3. Read data according to the state of applying. If the `ApplyIndex` greater
   than or eqaul to the the coresponding latest `CommitIndex`, the request can
   be processed, rather than will push to the pending queue until it can be
   processed.

**Note**: Currently, we only handled read requests on the leader, so we have a
few optimizations. One is the request can be processed if the
`apply_index_term` equal to the `current_term` because we only return success
after apply. The another one is the local read based on the lease. However, if
the follower received a read request, the read requests on the leader must go
through the normal flow of the `ReadIndex` policy to ensure linear consistency.
This is because the apply status of the follower not guaranteed to be
consistent with the leader, the apply status of the follower may newer than the
leader.

In order to distinguish the replica reading, we will add an option in the
request context. This option will mark whether to allow a replica to read,
other logic remains the smase as before

### ReadIndex API

In order to let the AP engine read data the same as *Consistent Replica Read.
TiKV will provide a special API, which response to the request with the latest
`CommitIndex`. AP engine may need to handle the request same as described above.

## Drawbacks

*Consistent Replica Reading* will be slightly slower than leader reading.
because of the extra RPC, so it's more useful for the heavy read request.

## Alternatives

### ReadIndex API II

To avoid the extra logic in the AP engine, the `ReadIndex` API responds to the
client whether it can start reading. The request message from the client:

```protobuf
    message ReplicaReadRequest {
        Context context = 1;
        apply_index =2;
    }
```

If the `ReadIndex` less than the `apply_index` in the request, TiKV will
responds `TRUE`, otherwise it responds `FLASE`. and the client should do retry
for a `False` response.
