## Summary

The proposed method for reducing the traffic of pushing safe ts is hibernated region optimization. This method can be especially beneficial for large clusters since it can save a significant amount of traffic.

The motivation behind this approach is that TiKV currently uses the `CheckLeader` request to push resolved ts, and the traffic it costs grows linearly with the number of regions. By utilizing hibernated region optimization, it is possible to reduce this traffic and improve overall performance.

## Motivation

TiKV pushes forward resolved ts by `CheckLeader` request, the traffic it costs grows linearly with the number of regions. When dealing with a large cluster that requires frequent safe TS pushes, it may result in a significant amount of traffic. Moreover, this is probably cross-AZ traffic, which is not free.

By optimizing inactive regions, it is possible to reduce traffic significantly. In a large cluster, many regions are not accessed and remain in a hibernated state.

Let's review the safe ts push mechanism.

1. The leader sends a `CheckLeader` request to the followers with a timestamp from PD, which carries the resolved timestamp pushed in previous step 3.
2. Follower response whether the leader matched.
3. If quorum of the voters are still in the current leader's term, update leader's `safe_ts`.

In step 1, the leader generates safe ts, and in the next round, the followers apply those timestamps. However, there is an "advance-ts-interval" gap between two step 1s, which results in a safe timestamp lag for the followers.

In other words, to ensure that a read operation with a 1-second staleness works well with a high hit ratio among followers, you need to set `resolved-ts.advance-ts-interval` to 0.5 seconds. This will double the traffic of pushing safe ts.

**We need a solution that applies the safe TS more efficiently.**

## Detailed design

The design will be described in two sections: the improvement of active regions and a new mechanism for hibernated regions.

### Protobuf

We still need `CheckLeader` request to confirm the leadership. But with some additional fields.

```diff
message CheckLeaderRequest {
    repeated LeaderInfo regions = 1;
    uint64 ts = 2;
+   uint64 store_id = 3;
+   repeated uint64 hibernated_regions = 4;
}

message CheckLeaderResponse {
    repeated uint64 regions = 1;
    uint64 ts = 2;
+   repeated uint64 failed_regions = 3;
}
```

To apply safe ts for valid leaders as soon as possible, instead of waiting for the next round of advancing resolved timestamps, we need to send another `ApplySafeTS` request. The `ApplySafeTS` request is usually small, the traffic caused by it can be ignored.

```protobuf
message CheckedLeader {
    uint64 region_id = 1;
    ReadState read_state = 2;
}

message ApplySafeTsRequest {
    uint64 ts = 1;
    uint64 store_id = 2;
    repeated uint64 unsafe_regions = 3;
    repeated CheckedLeader checked_leaders = 4;
}

message ApplySafeTsResponse {
    uint64 ts = 1;
}
```

### Active Regions

The `CheckLeaderResponse` respond with the regions that pass the Raft safety check. The leader can then push its `safe_ts` for those regions. Since most regions will pass the safety check, it is not necessary to respond with the IDs of all passing regions. Instead, we can respond with only the IDs of regions that fail the safety check.

Another optimization is that we can confirm the leadership if the leader lease is hold, by calling Raft's read-index command. But this will involve in the Raftstore thread pool, more CPU will be used by this.

### Inactive Regions

Here we list the active regions without writing to inactive regions. In the future, TiKV will deprecate hibernated regions and merge small regions into dynamic ones. If this happens, inactive regions will not be a problem. However, for users who do not use dynamic regions, this optimization is still required.

To save traffic, we can push the safe timestamps of inactive regions together without sending the region information list. The `ts` field in `CheckLeaderRequest` is only used to build the relationship between the request and response, although it's fetched from PD. Ideally, we can push the safe timestamps of inactive regions using this `ts` value. Additionally, we can remove inactive regions from `CheckLeaderRequest.regions`. Modify `CheckLeaderRequest` as follows.

We only send the IDs for inactive regions. In the most ideal situation, both `LeaderInfo` and the region ID in the response are skipped, reducing the traffic from 64 bytes to 8 bytes per region.

One more phase is required to apply the safe ts, because in check leader process, the follower cannot decide whether the request is from a valid leader, so it keep the safe ts in it's memory and wait for apply safe ts request.

```diff
pub struct RegionReadProgressRegistry {
    registry: Arc<Mutex<HashMap<u64, Arc<RegionReadProgress>>>>,
+   checked_states: Arc<Mutex<HashMap<u64, CheckedState>>>,
}

+ struct CheckedState {
+    safe_ts: u64,
+    valid_regions: Vec<u64>,
+ }
```

To save traffic, `ApplySafeTsRequest.unsafe_regions` only contains the regions whose leader may be changed. In the ideal case, this request is small because there is almost no unsafe regions.

#### Safety

Safety is guaranteed as long as the safe ts is generated from a **valid leader**. By decoupling the pushing of safe ts into two phases, the leader peer can ensure that the leadership is still unchanged before safe ts is generated.

For inactive regions, only the region IDs are sent. If the term has changed since the last active region check, the follower will respond with a check failure. When the leader receives the `CheckLeaderResponse`, and the inactive region ID is not in `CheckLeaderResponse.failed_regions`, it means that the terms of the leader and follower are both unchanged. After checking leadership for inactive regions, it's safe to push the safe timestamps from the leader. Additionally, since there are no following writes in inactive regions, the `applied_index` check is unnecessary.

#### Implementation

To implement safety checks for inactive regions, we record the term when we receive active region checks. We then compare the recorded term with the latest term when we receive a region ID without leader information, which indicates that the region is inactive. If the terms do not match, the leader must treat it as an active region next time.

## Drawbacks

This RFC make the resolved ts management more complex.

If there are too many active regions, more CPU will be consumed.

## Unresolved questions
