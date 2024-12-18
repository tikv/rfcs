# Cache read index resposnses

## Summary
The purpose of this design is to reduce the CPU uitlization on the leader by caching the response of read index requests from followers. By avoiding the need to send these requests when there are no pending writes on the leader at a specific read timestamp, we can significantly reduce the CPU overhead caused by expensive read index RPCs.

## Motivation

In a cloud environment, customers often prefer to perform follower reads within the same AZ to avoid incurring cross-AZ network costs. However, each follower read requires sending a read index request to the leader to check for any pending write locks in memory. These RPC calls can be quite expensive and can significantly increase CPU utilization, especially in read-heavy traffic scenarios. This increase in CPU cost eliminates the cost savings achieved by avoiding cross-AZ traffic.
To mitigate this issue in read-heavy environments, it is not necessary to send a read index request for every single follower read request. This is because, in 99% of cases, there are no pending locks. 

## Detailed design

We introduce a new concept last-read-index-ts. last-read-index-ts is updated with the read-ts when there is no memory lock in that region and apply_index is greated than commit_index of leader.  
In this proposal, response of read index messsages can be used to advance last-read-index-ts in follower if there are no pending pre writes on leader. 
Ref counters per region need to be added in the leader to keep track of total number of pending locks and lock status need to be included in a raft read index response. These are the detailed steps on how a follower read works
- Follower reads will initially compare the last-read-index-ts with the timestamp in the request, and only if the request timestamp is greater than the last-read-index-ts, a read index request will be sent.
- Read index request on leader check the lock reference counter and send its status through raft message in read index response
- Follower read response call update last-read-index-ts if there are no locks at this timestamp in leader.

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
### Code changes

Changes on follower
```
fn propose_raft_command {
    RequestPolicy::ReadIndex => {
        // if read-ts > last-read_index-ts than send read index message to leader
    }
}

fn apply_reads {
    let read_index_ctx = ReadIndexContext::parse(state.request_ctx.as_slice()).unwrap();
    // update last_read_indx_ts if apply_index > leader commit_index and there are no memory locks
    if read_index_ctx.memory_lock == false && 
        self.ready_to_handle_unsafe_replica_read(state.index) && 
        self.read_progress.last_read_indx_ts.load(Ordering::SeqCst) < start_ts {
        
        self.read_progress
            .last_read_index_ts
            .store(start_ts, Ordering::SeqCst);
    }
}

```

Changes on leader
```
impl ReadIndexObserver for ReplicaReadLockChecker {
    fn on_step(&self, msg: &mut eraftpb::Message, role: StateRole) {
        // set memory lock to false if in memory lock on a region is false
    }

impl<EK, ER> Peer<EK, ER>
where
    EK: KvEngine,
        ER: RaftEngine,
        {
            pub fn step<T>() -> Res {
                // if memory lock is set to false in on_step()
                rctx.memory_lock = false
        }
```

## Drawback
Based on the experiements it helps in reducing the read index requests by 70% if ```tidb_low_resolution_tso``` is set to true and ```tidb_low_resolution_tso_update_interval``` is set to 100 ms. However, there can be no cache hit if ```tidb_low_resolution_tso``` is set to false. Cache hit is proportional to ```tidb_low_resolution_tso_update_interval``` setting. 

## Alternatives

- An alternative approach would be to use batching to send fewer read index requests to the replica. However, batching can increase the p50 latency and result in higher network traffic compared to caching. Caching can tolerate smaller blips on leader unavailable depending on ```tidb_low_resolution_tso_update_interval``` , while batching may not be as reliable.
