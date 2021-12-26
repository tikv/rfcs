# RFC: Incremental analyze table


## Summary

This proposal introduce a method to store some result of analyze request from TiDB in TiKV Storage, so that we do not calculate the whole table when we run `analyze table` next time.

## Motivation

The initial motivation was to reduce the impact of `analyze table` on the cluster. And then I found that most of the calculations are not needed.

- In produce environment, most of regions will not be update in a long time.
- The analyze request will cost a lot of CPU resource but the result data of it is very small (In most cases, no more than 0.1% of the total data).
- We can easily know whether the data of a certain region has been changed.

So we can cache the data in TiKV, which is the result of analyze. 

## Detailed design

```protobuf
message StatisticCache {
    bytes data = 1;
    uint64 applied_index = 2;
}

message RaftSnapshotData {
    metapb.Region region = 1;
    uint64 file_size = 2;
    repeated KeyValue data = 3;
    uint64 version = 4;
    SnapshotMeta meta = 5;
    StatisticCache statistic_cache = 6;
}
```

* When a TiKV receives an analyze request it will check the cache and applied index of this region. If the cache data exists and `applied_index` of the cache equals to the current `applied_index` of this region, it means that the region has never write any user data or split.
* This data will be store by raft protocol. So it will also be send to a new peer when the member configure of raft members changes.

## Future Works

The current region-size of TiKV is usually set to '100MB'. If this size increases to be larger in the future, we may need to maintain the analyze results for each sub-range in a region.


