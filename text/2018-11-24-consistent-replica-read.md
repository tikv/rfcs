# Summary
`Consistent Replica Read` supports consistent read on the follower replica or 
learner replica, which can reduce the read load on the leader. `Consistent 
Replica Read` also useful in OLAP scenario or backup scenario.

The key to achieve consistent read on the replica is `ReadIndex`, which 
introduced in chapter 6 of the (raft 
thesis)[https://ramcloud.stanford.edu/~ongaro/thesis.pdf].

# Motivation

Currently, All read requests are handled on the leader, load balance only by 
`balance leader` operator, there are lots of limitations. On one hand, we hope 
a way to improve read throughput and divert load away from the leader. and On 
the other hand, hope some nodes support read-only service and handle some heavy 
analytics queries.

# Detail design

## Consistent Replica Read
As summary said, the key to implementing `Consistent Replica Read` is 
`ReadIndex`, which already implemented in our 
(raft-rs)[http://github.com/pingcap/raft-rs] library. For now, some read 
requests already use `ReadIndex` policy to handle, as the following steps.

1. Determine the policy of the read request. If the request is required to 
check the quorum, the replica is a leader and expired the lease, or the replica 
is a follower or a learner, should use `ReadIndex` policy.
2. Got `ReadIndex` from the raft library. Assign a `uuid` for the request and 
invoke the `read_index` API of the raft library with the data that encoded by 
`uuid`. then the raft state machine will notify our apply state machine with 
the `read_status` through the `Ready` message. and we can get the `ReadIndex` 
and `uuid` from the `read_status`.
3.  Read data according to the state of applying. If the `apply_index_term` 
equal to the `current_term`, the request can be processed, rather than will 
push to the pending queue.
**Note**: For the leader, the request can be processed if the 
`apply_index_term` equal to the `current_term`. For the follower and the 
learner, should let `apply_index` greater than or equal to the `ReadIndex`.

In order to distinguish the replica reading, will add an option in the request 
context. the option mark whether to allow the replica to read. the other logic 
same as before.

## ReadIndex API
In order to let the AP engine can read data same as `Consistent Replica Read`. 
TiKV will provide a special API, which response the request with the 
`ReadIndex`. AP engine may need to handle the request same as described above.

# Drawbacks
Consistent Replica reading will slightly slower than leader reading. because of 
the extra RPC. So it's more useful for the heavy read request.

# Alternatives
## ReadIndex API
For avoiding the extra logic in the AP engine, the ReadIndex API response the 
client whether it can start reading. The request message from the client:

    ```protobuf
    message ReplicaReadRequest {
        Context context = 1;
        apply_index =2;
    }
    ```
If the `ReadIndex` less than the `apply_index` in the request, TiKV will 
responses `TRUE`, ranther than `FLASE`. and the client should do retry for the 
`False` response.
