# Summary

Support `BatchSplit` feature that split one Region into multiple Regions at a time if the size is large enough. This includes modifications of both TiKV and PD. For TiKV, every round of split-check produces not only one split key but multiple split keys and change inner split related interface into batch style. For PD, add RPC `AskBatchSplit` and `ReportBatchSplit`  to permit TiKV asking for `region_id` and `peer_id` in batch.

# Motivation

Current split only split one Region at a time. It may be very slow when sequential write is too fast, namely, split speed can not keep up with write speed. Slow split can lead to large region. In this case, if a snapshot is triggered, it will occupy a lot of IO and make everything slow. Also, large region is hard for hotspot schedule, so it makes performance even worse.

# Detailed design

## RPC interface

### PD

```protobuf
service PD {
	// ...
	
    rpc AskSplit(AskSplitRequest) returns (AskSplitResponse) {
        // Use AskBatchSplit instead.
        option deprecated = true;
    }    
    rpc ReportSplit(ReportSplitRequest) returns (ReportSplitResponse) {
        // Use ResportBatchSplit instead.
        option deprecated = true;
    }
    rpc AskBatchSplit(AskBatchSplitRequest) returns (AskBatchSplitResponse) {}
    rpc ReportBatchSplit(ReportBatchSplitRequest) returns (ReportBatchSplitResponse) {}
}

message AskBatchSplitRequest {
    RequestHeader header = 1;
    metapb.Region region = 2;
    uint32 split_count = 3;
}

message SplitID {
    uint64 new_region_id = 1;
	repeated uint64 new_peer_ids = 2;
}

message AskBatchSplitResponse {
    ResponseHeader header = 1;
    repeated SplitID ids = 2;
}

message ReportBatchSplitRequest {
    RequestHeader header = 1;
	repeated metapb.Region regions = 2;
}

message ReportBatchSplitResponse {
	ResponseHeader header = 1;
}
```

Add  `AskBatchSplit`  to replace `AskSplit` , it is called when TiKV produces some split keys for one Region and asks PD to allocate new `region_id` and `peer_id` for that Region. `split_count`  in `AskBatchSplitRequest` indicates the number of Region to be generated, and `AskBatchSplitResponse` returns all new allocated ids to TiKV.

Add  `ReportBatchSplit` to replace `ReportBatchSplit`, it is called when TiKV finish splitting Region. `ReportBatchSplitRequest` takes all metas of new generated Region for PD to update its related information.

For compatibility issue, the old interface is not deleted but set to deprecated. 

### TiKV

```protobuf
message SplitRequest {
    // ...
    // Will be ignored in batch split, use `BatchSplitRequest::right_derive` instead.
    bool right_derive = 4 [deprecated=true];
}

message BatchSplitRequest {
    repeated SplitRequest requests = 1;
    // If true, the last region derive the origin region_id, 
    // other regions use new ids.
    bool right_derive = 2;
}

message BatchSplitResponse {
    repeated metapb.Region regions = 1;
}

enum AdminCmdType {
    // ...
    Split = 2 [deprecated=true];
    // ...
    BatchSplit = 10;
}

message AdminRequest {
    // ...
    SplitRequest split = 3 [deprecated=true];
    // ...
    BatchSplitRequest splits = 10;
}

message AdminResponse {
    // ...
    SplitResponse split = 3 [deprecated=true];
    // ...
    BatchSplitResponse splits = 10;
}
```

Add a new admin command type `BatchSplit`  with related request and response. `BatchSplitRequest` wraps multiple `SplitRequest` along with `right_derive` which invalidates the `right_derive` in each `SplitRequest`.

When in rolling upgrade process, new TiKVs are mixed up with old TiKVs, so old command type `Split` still needs to be preserved.

## Implementation in TiKV

### How to produce multiple split keys

This part mainly focus on `SplitChecker`. 

First of all, adjust trait to make it can return multiple split keys.

```rust
pub trait SplitChecker {
    // ...
    
    // before: fn split_key(&mut self) -> Option<Vec<u8>>
    fn split_keys(&mut self) -> Vec<Vec<u8>>;
    
    // before: fn approximate_split_key(&self, _: &Region, _: &DB) -> Result<Option<Vec<u8>>> 
    fn approximate_split_keys(&self, _: &Region, _: &DB) -> Result<Vec<Vec<u8>>> {
}
```

Then add one config `batch_split_limit` to limit the number of produced split keys at a time. If it is unlimited, for once split check, it scans all over the Region's range, and in some extreme case it would cause performance issue.

Now we have four split-checkers: half, key, size, table. SizeChecker and KeysChecker can be rewritten to produce multiple keys, and other checkers' logic stay unchanged. 

The general logic of SizeChecker and KeysChecker is similiar, the only difference of them is one splits Region based on size and the other splits Region based on the number of keys. So here we mainly describe the logic of SizeChecker:

- before: it scans key-value pairs in a Region's range sequentially to accumlate their size as `total_size` and stops once the size reachs to `region_max_size` or scans to the end of range. If  `total_size` is smaller than `region_max_size` at the end, checker wouldn't produce any split key; if not, it regards the very key at which `total_size`  reachs to `region_split_size` as split key.
- after: it scans key-value pairs in a Region's range sequentially to accumlate their size as `total_size` and stops once the size reachs to `region_split_size * (batch_split_limit-1) + region_max_size` or scans to the end of range. During the scan process, it reocrds the key every `region_split_size`  as split keys, but after finish scanning, it may discards the last split key if the size of rest Region doesn't over `region_max_size - region_split_size`. With this algorithm, if `batch_split_limit` is set to 1, TiKV can perfectly behaves as before that split without batch.

### Compatibility concern

The general process in raftstore changes a little, it mainly replaces `Split` with `BatchSplit`. But one thing should be noted, when rolling update PD version control will refuse `AskBatchSplit` request, thus split can't be performed during this process until all TiKV bump to new version. To let TiKV know whether `AskBatchSplit` fail reason is compatibility or not, we introduce a new error type for `ResponseHeader` :

```protobuf
enum ErrorType {
    // ...
    INCOMPATIBLE_VERSION = 5;
}
```

So once TiKV gets  `AskBatchSplitResponse`  with `ErrorType::INCOMPATIBLE_VERSION`, it uses original `AskSplit` instead of `AskBatchSplit`, and all following processes will degrade to original way. So original code path is not deleted.

### Approximate split key

What we said above can ease the problem, however scanning a large Region can also consumes a lot of time and CPU. Tests show that large region can still easily show up even with batch split implemented, although split is speeded up.

When a Region becomes large enough, it's more practical to divide it into smaller chunks quickly. This can be achieved via size estimation, which can be calculated from SST properties. Although it may not be accurate enough, it's okay for a large Region.

So if the size of Region is larger than `region_max_size * batch_split_limit * 2`, TiKV will use approximate way to produce split key. The approximate way is quite similar to the algorithm we describe above, but to estimate TiKV uses approximate size of the Region and the number of keys in the Region's range to calculate the average distance between two SST property keys, and produces a split key every `region_split_size / distance` keys.

# Drawbacks

- When use approximate way, Region may split into several disproportion Regions due to size estimation.

# Alternatives

None


# Unresolved questions

A large Region is usually more emergent to be split, so we can change the split check queue from a naive FIFO queue to a priority queue so that large Region can be split early and quickly.
