## Summary

The proposed method for reducing the traffic of pushing safe ts is hibernated region optimization. This method can be especially beneficial for large clusters since it can save a significant amount of traffic.

The motivation behind this approach is that TiKV currently uses the `CheckLeader` request to push resolved ts, and the traffic it costs grows linearly with the number of regions. By utilizing hibernated region optimization, it is possible to reduce this traffic and improve overall performance.

## Motivation

TiKV pushes forward resolved ts by `CheckLeader` request, the traffic it costs grows linearly with the number of regions. When dealing with a large cluster that requires frequent safe TS pushes, it may result in a significant amount of traffic. Moreover, this is probably cross-AZ traffic, which is not free.

By optimizing hibernated regions, it is possible to reduce traffic significantly. In a large cluster, many regions are not accessed and remain in a hibernated state.

## Detailed design

The design will be described in two sections: the improvement of active regions and a new mechanism for hibernated regions.

### Active Regions

```diff
message RegionEpoch {
    uint64 conf_ver = 1;
    uint64 version = 2;
}

message LeaderInfo {
    uint64 region_id = 1;
    uint64 peer_id = 2;
    uint64 term = 3;
    metapb.RegionEpoch region_epoch = 4;
    ReadState read_state = 5;
}

message ReadState {
    uint64 applied_index = 1;
    uint64 safe_ts = 2;
}

message CheckLeaderRequest {
    repeated LeaderInfo regions = 1;
    uint64 ts = 2;
}

message CheckLeaderResponse {
-   repeated uint64 regions = 1;
+   repeated uint64 failed_regions = 1;
    uint64 ts = 2;
}
```

The above code is the suggested changes for `CheckLeader` protobuf.

The `CheckLeaderResponse` will respond with the regions that pass the Raft safety check. The leader can then push its `safe_ts` for those regions. Since most regions will pass the safety check, it is not necessary to respond with the IDs of all passing regions. Instead, we can respond with only the IDs of regions that fail the safety check.

### Inactive Regions

Here we name the regions without writing to inactive regions. In the future TiKV will deprecate hibernated regions and merge the small regions into dymanic regions, if so the inactive regions won't be a problem, but for users that disable dynamic regions, this optimization is still required.

To save traffic, we can push the safe timestamps of inactive regions together without sending the region information list. The `ts` field in `CheckLeaderRequest` is only used to build the relationship between the request and response, although it's fetched from PD. Ideally, we can push the safe timestamps of inactive regions using this `ts` value. Additionally, we can remove inactive regions from `CheckLeaderRequest.regions`. Modify `CheckLeaderRequest` as follows.

We only send the IDs for inactive regions. In the most ideal situation, both `LeaderInfo` and the region ID in the response are skipped, reducing the traffic from 64 bytes to 8 bytes per region.

```diff
message CheckLeaderRequest {
    repeated LeaderInfo regions = 1;
    uint64 ts = 2;
    uint64 peer_id = 3;
+   repeated uint64 inactive_regions = 4;
}
```

One more phase is required to apply the safe ts, because in check leader process, the follower cannot decide whether the request is from a valid leader, so it keep the safe ts in it's memory and wait for apply safe ts request.

```protobuf
message ApplySafeTsRequest {
    uint64 ts = 1;
    repeated uint64 unsafe_regions = 2;
}

message ApplySafeTsResponse {
    uint64 ts = 1;
}
```

To save traffic, `ApplySafeTsRequest.unsafe_regions` only contains the regions whose leader may be changed. In the ideal case, this request is small because there is almost no unsafe regions.

#### Safety

Safety is guaranteed as long as the leader remains unchanged. By decoupling the pushing of safe ts into two phases, the leader peer can ensure that the leadership is still unchanged before safe ts is generated. Only the region IDs of inactive regions are sent, but we can still confirm that the leader has not changed for a follower if both the leader's term and the follower's term remain unchanged. Because the safe ts is applied only after the leadership is confirmed, correctness will not be compromised.

#### Implementation

To implement safety checks for hibernated regions, we record the term when we receive leader information. We then compare the recorded term with the latest term when we receive a region ID without leader information, which indicates that it is hibernating in the leader. If the terms do not match, we must send the `LeaderInfo` for this region the next time.

## Drawbacks

This RFC make the resolved ts management more complex.

## Unresolved questions
