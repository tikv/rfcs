## Summary

The proposed method for reducing the traffic of pushing safe ts is hibernated region optimization. This method can be especially beneficial for large clusters since it can save a significant amount of traffic.

The motivation behind this approach is that TiKV currently uses the `CheckLeader` request to push resolved ts, and the traffic it costs grows linearly with the number of regions. By utilizing hibernated region optimization, it is possible to reduce this traffic and improve overall performance.

## Motivation

TiKV pushes forward resolved ts by `CheckLeader` request, the traffic it costs grows linearly with the number of regions. When dealing with a large cluster that requires frequent safe TS pushes, it may result in a significant amount of traffic. Moreover, this is probably cross-AZ traffic, which is not free.

By optimizing hibernated regions, it is possible to reduce traffic significantly. In a large cluster, many regions are not accessed and remain in a hibernated state.

## Detailed design

The design will be described in two sections: the improvement of awake regions and a new mechanism for hibernated regions.

### Awake Regions

```diff
message RegionEpoch {
    uint64 conf_ver = 1;
    uint64 version = 2;
}

message LeaderInfo {
    uint64 region_id = 1;
-   uint64 peer_id = 2;
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
+   uint64 peer_id = 3;
}

message CheckLeaderResponse {
-   repeated uint64 regions = 1;
+   repeated uint64 failed_regions = 1;
    uint64 ts = 2;
}
```

The above code is the suggested changes for `CheckLeader` protobuf.

The `peer_id` field is used to check whether the request leader is the same as the leader in the follower peer. However, `LeaderInfo.peer_id` is redundant, as the leaders in the same peer share the same `peer_id`. Therefore, we can move it to `CheckLeaderRequest.peer_id` to avoid duplicated traffic. **Unfortunately, we must keep this field even if it is unused, to maintain compatibility with protobuf.**

The `CheckLeaderResponse` will respond with the regions that pass the Raft safety check. The leader can then push its `safe_ts` for those regions. Since most regions will pass the safety check, it is not necessary to respond with the IDs of all passing regions. Instead, we can respond with only the IDs of regions that fail the safety check.

### Hibernated Regions

To save traffic, we can push the safe timestamps of hibernated regions together without sending the region information list. The `ts` field in `CheckLeaderRequest` is only used to build the relationship between the request and response, although it's fetched from PD. Ideally, we can push the safe timestamps of hibernated regions using this `ts` value. Additionally, we can remove hibernated regions from `CheckLeaderRequest.regions`. Modify `CheckLeaderRequest` as follows.

```diff
message CheckLeaderRequest {
    repeated LeaderInfo regions = 1;
    uint64 ts = 2;
    uint64 peer_id = 3;
+   repeated uint64 hibernated_regions = 4;
}
```

We only send the IDs for hibernated regions. In the most ideal situation, both `LeaderInfo` and the region ID in the response are skipped, reducing the traffic from 8 bytes to 1 byte per region.

#### Safety

As long as the leader remains unchanged, safety is guaranteed. Specifically, if the terms of the quorum peers are not changed, we can assume that the leader is also unchanged. The `CheckLeaderRequest` is sent from the leader to the follower. If the terms of both the leader and the follower are unchanged, the condition "terms of the quorum peers are not changed" is met, and we believe there is no new leader.

#### Implementation

To implement safety checks for hibernated regions, we record the term when we receive leader information. We then compare the recorded term with the latest term when we receive a region ID without leader information, which indicates that it is hibernating in the leader. If the terms do not match, we must send the `LeaderInfo` for this region the next time.

## Drawbacks

This RFC make the resolved ts management more complex.

## Unresolved questions
