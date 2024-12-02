# Cache read index resposnses

## Summary
The purpose of this design is to minimize the CPU load on the leader by caching the response of read index requests from followers. By avoiding the need to send these requests when there are no pending writes on the leader at a specific read timestamp, we can significantly reduce the CPU overhead caused by expensive read index RPCs.

## Motivation

In a cloud environment, customers often prefer to perform follower reads within the same AZ to avoid incurring cross-AZ network costs. However, each follower read requires sending a read index request to the leader to check for any pending write locks in memory. These RPC calls can be quite expensive and can significantly increase CPU utilization, especially in read-heavy traffic scenarios. This increase in CPU cost eliminates the cost savings achieved by avoiding cross-AZ traffic.
To mitigate this issue in read-heavy environments, it is not necessary to send a read index request for every single follower read request. This is because, in 99% of cases, there are no pending locks. 

## Detailed design

Safe TS in follower is used to determine the minimum timestamp that guarantees the consistent reads from replica. Currently, safe TS is advanced by advance-ts message from the leader. In this proposal, response of read index messsages can also be used to advance safeTS in follower if there are no pending pre writes on leader. 
Changes to advance safe ts from raft read index response is pretty straightforward and most of the existing logic in the stale read path will be reused. 
Ref counters per region need to be added in the leader to keep track of total number of pending locks and lock status need to be included in a raft read index response. These are the detailed steps
- Read index request check the lock reference counter and send its status through raft message in read index response
- Update safe ts if there are no locks at this timestamp in replica.
- Follower reads will initially compare the Safe TS with the timestamp in the request, and only if the request timestamp is greater than the Safe TS, a read index request will be sent.

This feature would be enabled by default and no new configuration will be added. 

### Protocol

A new field `memory_lock` will be added to `ReadIndexContext` to send the lock status to replica.

```
pub struct ReadIndexContext {
    pub id: Uuid,
    pub request: Option<raft_cmdpb::ReadIndexRequest>,
    pub locked: Option<LockInfo>,
    pub memory_lock: Option<u64>,
}
```

## Drawback
Based on the experiements it helps in reducing the read index requests by 70% if ```tidb_low_resolution_tso``` is set to true and ```tidb_low_resolution_tso_update_interval``` is set to 100 ms. However, there can be no cache hit if ```tidb_low_resolution_tso``` is set to false. Cache hit is proportional to ```tidb_low_resolution_tso_update_interval``` setting. 

## Alternatives

An alternative approach would be to use batching to send fewer read index requests to the replica. However, batching can increase the p50 latency and result in higher network traffic compared to caching. Caching can tolerate smaller blips on leader availability depending on ```tidb_low_resolution_tso_update_interval``` , while batching may not be as reliable.

