# RFC: Return TiKV to some certain version

## Summary

This proposal introduces a method to return TiKV to some certain version, which could be useful when doing some lossy recovery.

## Motivation

The initial motivation is to do lossy recovery on TiKV when deploying TiDB cluster across DCs with DR Auto-sync method.

When use DR Auto-sync method to deploy TiDB cluster in two DC and 

- there is a network isolation between primary and DR DC
- PD make the replica state of primary DC to be Async after a while
- the primary DC is down

Then the sync state of the cluster (from the DR DC's PD) is not dependable. In this case, data can be partially sent to the TiKV nodes in DR, and break the ACID consistency.

Though it is impossible to do lossless recovery in this situation, we can recover the data on TiKV nodes to a certain version (`resolved_ts` in this case), which is ACID consistency.

## Detailed design

Basically, we need a method to return TiKV to a certain version (timestamp), we should add a new RPC to kvproto:

```protobuf
message FlashBackRequest {
    Context context = 1;
    uint64 checkpoint_ts = 2;
}

message FlashBackResponse {  
}

service Tikv {
		// …
		rpc FlashBack(kvrpcpb.FlashBackRequest) returns (kvrpcpb.FlashBackResponse) {}
}
```

When a TiKV recieve `FlashBackRequest`, it will create a `FlashBackWorker` instance and start the work.

There are several steps to follow when doing the flash back:

### 1. Wait all committed raft log are applied

We can just polling each region's `RegionInfo` in a method similar with [`Debugger::region_info`](https://github.com/tikv/tikv/blob/789c99666f2f9faaa6c6e5b021ac0cf7a76ae24e/src/server/debug.rs#L188) does, and wait for `commit_index` to be equal to `apply_index`.

### 2. Clean WriteCF and DefaultCF
 W
Then we can remove all data created by transactions which commited after `checkpoint_ts`, this can be done by remove all `Write` records which `commit_ts > checkpoint_ts`  and corresponding data in `DEFAULT_CF` from KVEngine, ie.

```rust
fn scan_next_batch(&mut self, batch_size: usize) -> Option<Vec<(Vec<u8>, Write)>> {
        let mut writes = None;
        for _ in 0..batch_size {
            if let Some((key, write)) = self.next_write()? {
                let commit_ts = Key::decode_ts_from(keys::origin_key(&key));
                let mut writes = writes.get_or_insert(Vec::new());
                if commit_ts > self.ts {
                    writes.push((key, write));
                }
            } else {
                return writes;
            }
        }
        writes
}

pub fn process_next_batch(
        &mut self, 
        batch_size: usize,
        wb: &mut RocksWriteBatch,
) -> bool {
        let writes = if let Some(writes) = self.scan_next_batch(batch_size) {
           writes
  			} else {
           return false;
  			}
        for (key, write) in writes {
            let default_key = Key::append_ts(Key::from_raw(&key), write.start_ts).to_raw().unwrap();
            box_try!(wb.delete_cf(CF_WRITE, &key));
            box_try!(wb.delete_cf(CF_DEFAULT, &default_key));
        }
  			true
}
```

Note we scan and process data in batches, so we can report the progress to the client.

### 3. Resolve locks once

After remove all the writes, we can promise we won't give the user ACID inconsistency data when processing read, however, there might be still locks in lock cf which block some user's operation.

So we should do resolve locks on all data once.

This lock resolving process is just like the one used by GC, with safepoint ts equals to current ts.

## Drawbacks

This is a **very** unsafe operation, we should prevent the user call this on normal cluster.

## Alternatives

- We can also clean the lock cf in step2 instead of do resolve locks.
