# Builtin GC worker

## Summary

1. Move TiKV MVCC GC worker from TiDB into PD and make it general for use without TiDB.

2. Support different safepoint for various ranges.

## Motivation

1. GC worker is an important component for TiKV that deletes outdated MVCC data so as to not explode the storage. But currently, the GC worker is implemented in TiDB, which makes TiKV not usable without TiDB.

2. Flashback is a feature under design for TiDB which allows querying SQL using a specified timestamp. By specifying GC deadline for different tables we can enlarge the use case that allowing arbitrary early timestamp (currently 10 mins) in flashback as long as a proper GC deadline is set for the table.

## Background

According to the [documentation](https://docs.pingcap.com/tidb/stable/garbage-collection-overview) for the **current** GC worker, the GC worker processes the following steps in a GC cycle:

1. Calculate the minimal timestamp among all living SQL sessions, earlier than which are safe to GC, called `Session Safepoint`. Then calculate the `Service Safepoint` for id `gc_worker` by `min(Session Safepoint, now - gc_life_time)`.
2. [Push](https://github.com/pingcap/kvproto/blob/8ecb5e46d7f5f7952a1a8d262b54f61dc8de1ef3/proto/pdpb.proto#L73) the `gc_worker Service Safepoint` to PD. At the meanwhile, other tools (e.g. CDC, BR) has also pushed their `Service Safepoint` for minimal snapshot timestamp requirement.
3. Get the minimal `Service Safepoint` among all services from the response of step 2, which is `GC Safepoint`.
4. Resolve all locks whose timestamp is earlier than the `GC Safepoint`.
5. [Push](https://github.com/pingcap/kvproto/blob/8ecb5e46d7f5f7952a1a8d262b54f61dc8de1ef3/proto/pdpb.proto#L71) the `GC Safepoint` to PD.

After the `GCSafepoint` being uploaded, TiKV will pull it from PD by interval, then TiKV will automatically delete all records earlier than `GCSafepoint`.

Yet another tricky part is that TiDB has a shortcut to GC a range of keys after `DROP TABLE` or `DROP INDEX`, etc: parallel with step 3 and step 4, TiDB will directly invoke `UnsafeDeleteRange` to TiKV which will delete the range of key-values regardless of locks or timestamp. To make it safe, TiDB will guarantee that the deleted range will never be used again.

## Detailed design

Four new concepts will be introduced by this proposal:

1. `Safeline`

    Similar to `Safepoint`, `Safeline` tells after which timestamp is safe to read and before which is safe to GC, but it will no longer be globally equal among the whole key range but instead separated by key ranges so as to differently push safepoint forward for different key ranges. In this case, safepoint will be a special cased `Safepoint` who ranges across the whole key space.

2. `GC Barrier`

    PD guarantees that using a timestamp later than the `GC Barrier` to read and write on the range within the key range of `GC Barrier` will always be valid. TiKV users (e.g. TiDB / CDC / Client) can put GC Barriers into PD so as to make sure the data they are using will not be GC.

    ![GC Barrier](../media/gc-barrier.png)

3. `Transaction Safeline`

    TiKV and TiFlash guarantee that no read or write request should success using the timestamp earlier than (above) `Transaction Safeline`. PD calculates the timestamp for each key range of `Transaction Safeline` by `min(GC Barriers)` and then proposes the `Transaction Safeline` to TiKV and TiFlash via the `StoreHeartbeatResponse`. Therefore, TiKV and TiFlash will reject all requests with timestamp earlier than (above) the `Transaction Safeline`. In the next following `StoreHeartbeatRequest` TiKV and TiFlash will report their latest `Transaction Safeline` to PD.

4. `GC Safeline`

    PD calculates `GC Safeline` by `min(Transaction Safeline Response) - 1` for each key range, resolves locks earlier than (above) `GC Safeline`, then synchronizes it to TiKV and TiFlash via `StoreHeartbeatResponse`. And finally, TiKV and TiFlash are safe to delete MVCC data earlier than (above) `GC Safeline`.

The graph below illustrates the GC process on an individual key.

![GC Worker](../media/gc-worker.png)

Note that both `Transaction Safeline` and `GC Safeline` must be non-regressive on TiKV, TiFlash and PD.

According to the design above, `gcpb.proto` will be added:

```proto
// gcpb.proto

message Safeline {
    repeated bytes split_keys = 1;
    // The item count of `range_timestamp` should be exactly one more than `split_keys`, e.g.,
    // a split key 'a' generates two key ranges: `..'a'` and `'a'..`, then `range_timestamp`
    // should specify two safepoints for two key ranges.
    repeated uint64 range_timestamp = 2;
}

message SafelineStatus {
    uint64 txn_safeline_version = 1;
    uint64 gc_safeline_version = 2;

    // Safeline will be set only if it's in `StoreHeartbeatResponse` and the `safeline_version`
    // in `StoreHeartbeatRequest` is not the latest.
    bool has_txn_safeline = 3;
    Safeline txn_safeline = 4;
    bool has_gc_safeline = 5;
    Safeline gc_safeline = 6;
}

service GCWorker {
   rpc GetGCStatus(GetGCStatusRequest) returns (GetGCStatusResponse) {}
   rpc PutGCBarrier(SetGCBarrierRequest) returns (SetGCBarrierResponse) {}
}

message GCBarrier {
   bytes id = 1;
   // Set to 0 to delete barrier
   uint32 barrier_ttl_seconds = 2;

   // Two modes for a GCBarrier. Only one can be set.
   uint64 fixed_timestamp = 3;
   uint32 seconds_before = 4;

   bytes start_key = 5;
   bytes end_key = 6;
}

message GetGCStatusRequest {
   RequestHeader header = 1;
}

message GetGCStatusResponse {
   ResponseHeader header = 1;
   repeated GCBarriers gc_barriers = 2;
   SafelineStatus safeline_status = 3;
}

message PutGCBarrierRequest {
   RequestHeader header = 1;
   GCBarrier gc_barrier = 2;
}

message PutGCBarrierResponse {
   ResponseHeader header = 1;
}
```

Also, a few fields will be added to `pdpb.proto`:

```diff
  // pdpb.proto

  message StoreHeartbeatRequest {
      RequestHeader header = 1;
      StoreStats stats = 2;
  }

  message StoreStats {
      ...
+     gcpb.SafelineStatus safeline_status = 15;
  }

  message PutStoreResponse {
      ResponseHeader header = 1;
      replication_modepb.ReplicationStatus replication_status = 2;
+     gcpb.SafelineStatus safeline_status = 3;
  }

  message StoreHeartbeatResponse {
     ResponseHeader header = 1;
     replication_modepb.ReplicationStatus replication_status = 2;
     string cluster_version = 3;
+    gcpb.SafelineStatus safeline_status = 4;
  }
```

### Backward compatibility

#### Rolling update

During the rolling update, the GC worker should not function and the new TiDB should not try to be elected as a GC leader. Once the rolling update is completed, the GC worker in PD will start to work.

The RPC `GetGCSafePoint` and `UpdateServiceGCSafePoint`, which are used by the GC model before this proposal should still work for keep compatible with tools and they should be just a thin wrapper on top of the new `GetGCStatus` and `PutGCBarrier`.

`UpdateGCSafePoint` should only work before rolling update completes, once the rolling update finishes, PD will reject requests to this RPC.

#### TiDB unsafe delete range

In the model described above, we didn't specify when TiDB should unsafe delete range. Because the step of resolving lock is controlled by PD and unsafe delete range is only allowed after `GC Safepoint` is pushed ahead of the timestamp of `DROP TABLE`, TiDB has to passively listen to the `GC Safepoint` (fetch by interval), and perform unsafe delete range when appropriate.

## Reference

<https://docs.google.com/document/d/1jA3lK9QbYlwsvn67wGsSuusD1Dzx7ANq_vya384RBIg/edit#heading=h.rr3hcmc7ejb8>

<https://docs.pingcap.com/tidb/stable/garbage-collection-overview>

<https://www.cockroachlabs.com/docs/v20.1/configure-replication-zones#create-a-replication-zone-for-a-table>
