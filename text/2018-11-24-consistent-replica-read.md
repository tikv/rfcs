# Consistent Replica Read

## Summary

*Consistent Replica Read* supports consistent read on the follower replica or
learner replica, which can reduce the read load on the leader. *Consistent
Replica Read* is also useful in OLAP or backup scenarios.

The key to achieve consistent read on the replica is `ReadIndex`, which was
introduced in Chapter 6 of the [Raft
thesis](https://ramcloud.stanford.edu/~ongaro/thesis.pdf).

## Motivation

Currently, all read requests are handled on the leader, and load balancing is
performed only by the `balanceLeader` operator, which have lead to many
limitations. On one hand, we want a way to improve read throughput and divert
load away from the leader; on the other hand, we want some nodes to handle
read-only service and some of the heavy analytics queries.

## Detail design

### Replica Read

As indicated in the Summary section, the key to implementing *Consistent
Replica Read* is `ReadIndex`, which has already been implemented in our
[raft-rs](http://github.com/pingcap/raft-rs) library. For now, some read
requests are already handled using the `ReadIndex` policy. It follows these
steps:

1. Determine the policy of the read request. If the request is required to
   check the quorum, the replica is a leader with an expired lease, or the
   replica is a follower or a learner, it should be using the `ReadIndex`
   policy.
2. Get the latest `CommitIndex` from the raft library. Assign a `uuid` for the
   request and invoke the `read_index` API of the raft library with the data
   that was encoded by `uuid`. Then the raft state machine will notify our apply
   state machine with the `read_status` through the `Ready` message. And we can
   get the latest `CommitIndex` through the corresponding `uuid` from the
   `read_status`.
3. Read data according to the state of applying. If the `ApplyIndex` is greater
   than or equal to the corresponding latest `CommitIndex`, the request can be
   processed. Otherwise, it will be pushed to the pending queue until it can be
   processed.

**Note**: Currently, we only handle read requests on the leader, so we have a
few optimizations. One of them is that the request can be processed if
`apply_index_term` equals to the `current_term`, as we only return success
after apply. Another one is local read based on the lease. However, if the
follower receives a read request, the read requests on the leader must go
through the normal flow of the `ReadIndex` policy to ensure linear consistency.
This is because the apply status of the follower is not guaranteed to be
consistent with the leader, for instance, the apply status of the follower
may be newer than the leader.

In order to distinguish the replica reading process, we intend to add an
option in the request context, which will mark whether to allow a replica to
read. Other logics remains the same as before.

### ReadIndex API

In order for the AP engine to read data the same way as *Consistent Replica
Read*. TiKV will provide a special API, which responds to the request with the
latest `CommitIndex`. The AP engine may need to handle the request as described
above.

## Drawbacks

Due to extra RPC load, *Consistent Replica Reading* will be slightly slower than
leader reading. It's more suitable for scenarios with heavy read requests.

## Alternatives

### ReadIndex API II

To avoid extra logic in the AP engine, the `ReadIndex` API responds to the
client whether it can start reading. The request message from the client is
as shown below:

```protobuf
    message ReplicaReadRequest {
        Context context = 1;
        apply_index = 2;
    }
```

If the `ReadIndex` less than the `apply_index` in the request, TiKV will
responds `TRUE`, otherwise it responds `FALSE`. Then the client should retry
for a `FALSE` response.
