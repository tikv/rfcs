# Mark CF

- RFC PR: (to be filled)
- Tracking issue: (to be filled)

## Summary

Add a new CF (column family) `mark` to store `Lock` and `Rollback` records of transactions. Whenever a `Lock` or `Rollback` record is written to the write CF, also write a copy to the mark CF. Then, the `Lock` and `Rollback` records can be removed once they are not the latest version, so they won't affect the read performance a lot. 

## Motivation

Consider a use case: the user application always reads and locks a fixed key but never changes it:

```
txn = client.start_transaction()
txn.lock("k1")
v1 = txn.get("k1")
txn.put("k2", "v2")
txn.commit()
```

No matter it's an optimistic or pessimistic transaction, the transaction will write a new `Lock` record on `k1` in the write CF. If we repeats executing this transaction, there can be a lot of `Lock` records on `k1` that are newer than the effective `PUT` record, like:

```
k1@100  Lock
k1@99   Lock
k1@98   Lock
...
k1@11   Lock
k1@10   PUT   value = "v1"
```

It will be a disaster when we want to read the value of `k1`. If it's a point get, we can only seek to the latest version and waste a lot of effort calling `next` until we find an effective record. The performance of range scan is also degraded. Furthermore, the _current_ MVCC GC cannot remove these `Lock` records because they are newer than the latest `PUT` records. So, the performance will keep getting worse until the key is effectively modified.

## Design

`Lock` and `Rollback` records have no value in them. They are totally ignored during reading. If they are causing much trouble, why can't we remove them? Let's find out what role they perform now.

### Background

`Rollback` records exist to ensure the corresponding transaction must not succeed in committing. When acquiring a pessimistic lock of a key or prewriting a key, if a `Rollback` record _with the same `start_ts`_ exist on the key, it must not succeed.

We already have a collapsing mechanism to merge consecutive `Rollback` records. And it was invented back to the days when pessimistic transactions are not supported. Now, it's unlikely to have many `Rollback` records that affect read performance.

The `Lock` record is a bit more complicated. At the very beginning, when there is no pessimistic transaction or async-commit transaction, it is only used to check read-write conflicts, mostly for write-skew prevention. In pessimistic transactions, if a key is locked but not changed in the end, the pessimistic lock will be finally turned into a `Lock` record. In these cases, `Lock` records exist to cause write conflicts. If it happens to be the primary key of the transaction, it also marks the committed status. So, if the `Lock` record is only to cause write conflicts, it doesn't need to exist after any newer record is written. However, it is not true for the primary keys.

In async-commit transactions, every key is important to the status of the transaction. If a committed `Lock` record is removed or collapsed, we cannot tell whether the key is never prewritten before or it is a collapsed `Lock` record. This makes it impossible to resolve an async-commit lock. In other words, after async commit is supported, none of the `Lock` records can be collapsed by newer records.

To conclude, `Lock` and `Rollback` records are important in these two aspects:

- Transaction status (cannot be deleted until GC)
- Conflict check (can be removed after a new write record)

### Mark CF

Now, we know that we have to store the `Lock` records somewhere because they are vital for us to know the transaction status. But we don't want them to affect the reading performance. So, a new column family is a good idea. We call it the `mark` CF because it doesn't include any effective data. Instead, they are just marks of transaction status.

Besides `Lock` records, the `Rollback` records are also put into this CF because they are similar mark records. This also avoids some tricky problems when we have to overwrite some records in the write CF when writing a `Rollback` record. We'll talk about it later.

#### Format

Key: `{user_key}{start_ts}`

Value: `{write_type}{commit_ts}`

Unlike the write CF, we concatenate the start TS instead of the commit TS to the end of the user key. This is because we read the mark CF only when we consult the status of a key in the transaction, in which case we have the start TS of the transaction.

But it brings another problem. `Lock` and `Rollback` records are also important to conflict checks. It would be sad if we need an extra seek in the new CF for a conflict check. Is it possible to avoid the regression?

### `Lock` and `Rollback` records in the write CF

The solution to the extra seek is writing `Lock` or `Rollback` records to the write CF as usual. So, the conflict checks only need to seek the write CF for the latest version.

This means, we are always writing the records to both the write CF and the mark CF. This brings amplification, but is acceptable in most use cases. Because these records don't contain the value, it's unlikely that the `Lock` records occupies a significant part of the write flow.

As the mark CF satisfies the need of keeping transaction status, these `Lock` and `Rollback` records in the write CF are only for conflict checks. Therefore, once the records are not the latest version, we can safely remove them from the write CF.

**NB.** There are a few cases when the latest version is not enough for conflict checks. For example, `Rollback` records may be skipped when checking newer versions when prewriting non-pessimistic keys in pessimistic transactions. In this case, we also need to read the mark CF to confirm.

#### Checking the write CF early

Write CF records are typically written in the commit command, during which we don't read the write CF before. So, it will bring extra cost if we check whether the latest version can be removed in the commit command.

We can move the check forward to the conflict check process in acquiring pessimistic locks or prewriting. During the conflict check, we can see whether the latest record is a `Lock` or `Rollback` record. If so, we record its commit TS in the lock:

```rust
pub struct Lock {
    ...
    /// Commit TS of a `Lock` or `Rollback` record which is the latest version of the key
    pub latest_mark_ts: TimeStamp,
}
```

Then, when the lock is finally committed into a write record, we can delete the stale `Lock` or `Rollback` record without additionally reading the write CF again.

#### Overwriting the latest `Lock` or `Rollback` (to be discussed)

This is a tricky optimization and I really doubt if it is really worth.

If a new `Lock` record is written to the write CF while the latest version is also a `Lock` or `Rollback`, instead of removing the previous version, just overwrite that record and add a `real_commit_ts` to the record. When checking write conflicts, we should parse the value and check the real commit TS because the timestamp encoded into the key may be not accurate.

It may help reduce tombstones but breaks too many assumptions before.

### Reading the mark CF

Putting `Lock` and `Rollback` records in the mark CF does not introduce extra seek operations to the happy path of transactions. But it does when we talk about resolving locks.

Now, the records that represent transaction status spread in both the write CF and the mark CF.

In `CheckTxnStatus`, we need to read both CFs of the primary key for the status of the transaction.

In `CheckSecondaryKeys`, we need to check both CFs of all the given secondary keys to know whether some keys are already committed or rolled back.

And when prewrite raises an error or we are prewriting non-pessimistic keys in a retry, we also need the precise status of the key to guarantee idempotence. This also requires to read both the write and mark CFs.

Luckily, all of these don't happen frequently in production. The extra cost is not a big issue.

### No more overlapped rollback

This is an optional change. Overlapped rollback describes the case when the `start_ts` of a `Rollback` record equals to the `commit_ts` of another write record, the `Rollback` record will become a special flag in that write record. We have implemented it, but it was proved to be a tricky solution as it caused incompatibilities with the compaction filter and CDC mainly because it overwrites an existing write record. 

It is possible for us to get rid of it now. First, it's impossible for a normal write record to overwrite a protected `Rollback` in the write CF because we update the `max_ts` when writing a protected `Rollback`. So, we only need to consider overwriting by a new `Rollback`. Under such circumstances, we can just skip writing `Rollback` to the write CF. It does not affect conflict checking. And when we need the precise status of the transaction, the `Rollback` in the mark CF can also tell us.

Because the keys in the mark CF are encoded with `start_ts` and `start_ts` is the unique identifier of transactions, it is impossible to have key conflicts in the mark CF.

### Garbage collection

The records in the mark CF don't need to exist after all keys in the transaction are totally resolved. The client resolves all the locks before a certain timestamp before updating this timestamp as the safe point.

So, when TiKV is ready to do GC, all records in the mark CF whose `commit_ts` is less than the safe point can be deleted. It can be done in the compaction filter.

### Raftstore changes

- Support generating and ingesting snapshot for the mark CF
- Consider the keys and size of the mark CF in the split checker
- After removing the WAL of KV DB, the memtable needs to be flushed if it blocks the GC of the Raft logs.

### Upgrade from earlier versions

The new TiKV instances create the new mark CF when it starts up. During the rolling update process, some of the TiKV instances in the cluster may not have the new CF. So, we cannot write to the mark CF until all the TiKV instances in the cluster are upgraded to the new version. Otherwise, it will panic when applying the raft entry that writes to the mark CF.

We can use the feature gate to control the behavior. When we know from PD that the whole cluster has upgraded to a version supporting the mark CF, we start writing to the mark CF. 

Downgrade is not supported after some data is written to the mark CF.

### Compatibility with other components

#### CDC

In most cases, CDC can ignore the changes to the mark CF. But if we avoid writing an overlapped rollback, we can only know there is a rollback from the changes to the mark CF. So, generally we merge the changes to both the write CF and the mark CF and handle them uniformly in the CDC module in TiKV.

Deletions in the write CF can be ignored as usual.

#### BR

When BR takes a snapshot of TiKV, all the locks before the snapshot should be resolved. In this case, the records in the mark CF really don't matter.

BR can just ignore the mark CF.

