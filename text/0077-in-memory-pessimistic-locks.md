# RFC: In-memory Pessimistic Locks

- RFC PR: https://github.com/tikv/rfcs/pull/77
- Tracking Issue: https://github.com/tikv/tikv/issues/11452

## Motivation

Pessimistic locks are replicated via the Raft protocol and then applied to the Lock CF in RocksDB. Now, TiKV implements an optimization called "pipelined pessimistic lock", which returns the result to the user immediately after proposing the locking request successfully.

"Pipelined pessimistic lock" saves most of the latency spent on Raft replication. This optimization is mainly based on the fact that it is still safe if some pessimistic locks are lost at the time of committing the transaction.

We can take it a step further. It is feasible to only keep pessimistic locks in the memory of the leader and not replicate the locks via Raft. With appropriate handlings on region changes, the failure rate of transactions will not increase compared to "pipelined pessimistic lock".

This change expects to reduce disk write bandwidth by 20% and reduce the latency of pessimistic locking by 50% according to preliminary tests on the TPC-C workload.

## Detailed design

Here is the general idea:

- Pessimistic locks are written into a region-level lock table.
- Pessimistic locks are sent to other peers before a voluntary leader transfer.
- Pessimistic locks in the source region are sent to the target region before a region merge.
- On splitting a region, pessimistic locks are moved to the corresponding new regions.
- Each region has limited space for in-memory pessimistic locks.

Next, we will talk about them in detail.

### Lock table

The lock table is region-level and is backed by a simple hash table. 

The table is region-level because on region changes, we usually need to scan all pessimistic locks in a single region.

We don't have the requirement to scan pessimistic locks in a range ordered by key, so a hash table is sufficient instead of an ordered map.

We want to put transaction-related data together in `TxnExt`. Now, it contains `max_ts_sync_status` which is useful for async commit, and the newly added `PeerPessimisticLocks` protected by a `RwLock`.

```rust
pub struct Peer<EK, ER> {
    // ...
    txn_ext: Arc<TxnExt>,
}

pub struct TxnExt {
    max_ts_sync_status: AtomicU64,
    pessimistic_locks: RwLock<PeerPessimisticLocks>,
}
```

`PeerPessimisticLocks` contains a simple `HashMap` for the memory locks. And it contains `is_valid` marking its status and `term` and `version` indicating the metadata of the corresponding region. `size` is used to control the used memory. They will be explained later.

```rust
pub struct PeerPessimisticLocks {
    /// The table that stores pessimistic locks.
    ///
    /// The bool marks an ongoing write request (which has been sent to the raftstore while not
    /// applied yet) will delete this lock. The lock will be really deleted after applying the
    /// write request.
    map: HashMap<Key, (PessimisticLock, bool)>,
    size: usize,
     /// Whether the pessimistic lock map is valid to read or write. If it is invalid,
    /// the in-memory pessimistic lock feature cannot be used at the moment.
    pub is_valid: bool,
    /// Refers to the Raft term in which the pessimistic lock table is valid.
    pub term: u64,
    /// Refers to the region version in which the pessimistic lock table is valid.
    pub version: u64,
}

pub struct PessimisticLock {
    pub primary: Box<[u8]>,
    pub start_ts: TimeStamp,
    pub lock_ttl: u64,
    pub for_update_ts: TimeStamp,
    pub min_commit_ts: TimeStamp,
}
```

The transaction layer gets the lock table through the snapshot.

Here, we want to change a bit of the snapshot interface. Raftstore-specific data are provided by extensions. This unifies the way how the raftstore passed out extra information of a region.

```rust
pub trait Snapshot: Sync + Send + Clone {
    type Iter: Iterator;
    type Ext<'a>: SnapshotExt;

    fn ext(&self) -> Self::Ext<'_>;
}

pub trait SnapshotExt {
    fn get_data_version(&self) -> Option<u64>;

    fn is_max_ts_synced(&self) -> bool;

    fn get_term(&self) -> Option<NonZeroU64>;

    fn get_txn_extra_op(&self) -> ExtraOp;

    fn get_txn_ext(&self) -> Option<&Arc<TxnExt>>;
}

pub struct RegionSnapshot<S: Snapshot> {
    snap: Arc<S>,
    region: Arc<Region>,
    apply_index: Arc<AtomicU64>,
    term: Option<NonZeroU64>, // term is no longer returned via CbContext
    txn_extra_op: TxnExtraOp,
    txn_ext: Option<Arc<TxnExt>>,
}

pub struct RegionSnapshotExt<'a, S: Snapshot> {
    snapshot: &'a RegionSnapshot<S>,
}

impl<'a, S: Snapshot> SnapshotExt for RegionSnapshotExt<'a, S> {
    // ...
}
```

### Read and write pessimistic locks

After the lock table is retrieved, we can read and write pessimistic locks in the transaction layer.

Before processing an `AcquirePessimisticLock` command in the scheduler, we save the current `term` and `version` of the region. This will be checked later when writing locks to the table.

When loading a lock, we can read the in-memory lock table first. If no lock is found, then read the underlying RocksDB. When reading the lock table, we should check the `term` and `version` of the region. If `version` or `term` has changed, We should return `EpochNotMatch` or `StaleCommand` respectively.


And a new `Modify` variant is added. The `AcquirePessimisticLock` now adds `Modify::PessimisticLock` instead of a normal `Modify::Put`.

```rust
pub enum Modify {
    // ...
    PessimisticLock(Key, PessimisticLock),
}
```


With this policy, we need to check the `PeerPessimisticLocks` first. If `is_valid` is `false`, the feature is not usable probably because lock migration is in progress. And if the `term` or `version` is different from those before processing the command, the region must have changed and the lock table we fetched before processing is not the one we should write to. For simplicity, we can just continue sending the command to the raftstore to get the error and let the client retry when any check fails.

If the check passes, it is safe for us to write the pessimistic locks into the lock table. After this, we can return the response to the client without proposing anything at all.

For all writes involving the lock CF, the lock in the lock table should be cleared. For example, when a `Prewrite` command replaces a pessimistic lock with a 2PC lock, of course, this is a replicated write, we need to remove the pessimistic lock from the lock table after the write succeeds. It needs two steps to remove the lock. First, we mark the lock as `deleted` by changing the bool field in the lock table, and atomically send the write command to the raftstore. After the write command is finally applied, the lock is truly removed from the table, executing in the apply thread in the raftstore.

### Region leader transfer

If it is a voluntary leader transfer triggered by PD, we have the chance to transfer the pessimistic locks to the new leader to avoid unexpected transaction failures.

When a peer is going to transfer its leadership, the leader serializes current pessimistic locks **with no deleted marks** into bytes and modifies `is_valid` to `false`. Then, later `AcquirePessimisticLock` commands will fall back to proposing locks. In this way, we can guarantee the pessimistic locks either exist in the serialized bytes, or are replicated through Raft.

The serialized pessimistic locks will be sent to other peers through a Raft proposal. After this lock migration proposal is committed, the leader will continue the logic of in the current `pre_transfer_leader` method.

By default, a Raft message has a size limit of 1 MiB. We will guarantee that the total size of in-memory pessimistic locks in a single region will not exceed the limit. This will be discussed later.

If the transfer leader message is lost or rejected, we need to revert `is_valid` to `true`. But it is not possible for the leader to know. So, we have to use a timeout to implement it. That means, we can revert `is_valid` to `true` if the leader is still not transferred after some period. And if the leader receives the `MsgTransferLeader` response from the follower after the timeout, it should ignore the message and not trigger a leader transfer.

### Region merge

Before region merge, we should first set the `is_valid` to `false` to prevent future writings to the lock table. Then, record the current `proposed_index` and set a flag in the raftstore to forbid new write commands. After it applies to the recorded index, we can propose all the locks **including those with deleted marks** in the lock table at this time.

After proposing the locks, we can continue the original merge procedure, proposing `PrepareMerge`.

If the merge is rolled back, we can set `is_valid` of the source region back to `true`.

### Region split

After a region splits, if the parent peer is a leader, the newly split peer will campaign first to become a leader. So, the leader of all new regions are still located in the same TiKV unless there are network issues or the raftstore is too busy, in which case locks can be lost. Therefore, we only need memory operations to handle the case.

In `on_ready_split_region`, we first set `is_valid` to `false` and update the `version` of the lock table. Later pessimistic locks will either make a proposal or just fail, so we will not miss any lock.

Then, we iterate all locks **including those with deleted marks** and group them into new regions. After the locks are processed, we can increase `epoch` and set `is_valid` to `true`.

### Memory limit

To simplify handlings in region changes, we don't allow the total size of the pessimistic locks in a single region to be too large. The limit for each region is 512 KiB, matching the 1 MiB Raft message limit.

There should also be a global limit. The default size is the minimum of 1 GiB and 5% of the system memory.

It is easy for the lock writer to maintain the total size of pessimistic locks in a single region. If the memory limit is exceeded, we have to fall back to propose pessimistic locks.

### Compatibility

Pessimistic locks are sent via new fields in Raft messages. Only the upgraded TiKV instances can handle them while the old TiKV instances will ignore them. If the feature is unconditionally enabled, the pessimistic locks will be lost and the success rate of pessimistic transactions will drop during the rolling upgrade.

So this feature needs to be enabled after TiKV is fully upgraded. A possible approach is that TiKV gets the cluster version from PD and automatically turns it on through feature gate control.

The storage structure is not changed, so downgrading is feasible. However, before the downgrade, we must disable this feature first to avoid affecting the success rate of pessimistic transactions.

Ecosystem tools should not be affected by this optimization.

## Correctness

There are two main changes compared to "pipelined pessimistic lock": loss of pessimistic locks and lock migration.

### Loss of pessimistic locks

With "pipelined pessimistic lock", if a pessimistic lock is read, it will either be rolled back or turned into a 2PC lock. But this is different if pessimistic locks only exist in the memory. The pessimistic lock in the memory is readable, but it can be lost later due to various reasons like TiKV crash.

Luckily, this does not have much impact.

- Reading exactly the key of which pessimistic lock is lost is not affected, because the pessimistic lock is totally invisible to the reader.
- If a secondary 2PC lock is read while the primary lock is still in the pessimistic stage, the reader will call `CheckTxnStatus` to the primary lock:
  - If the primary lock exists, `min_commit_ts` of the lock is advanced, so the reader will not be blocked. **This operation must be replicated through Raft.** Otherwise, if the primary lock is lost, we may allow a smaller commit TS, breaking snapshot isolation.
  - If the primary lock is lost, `CheckTxnStatus` will do nothing until a lock is prewritten or the TTL is expired. The behavior is no different from before. There can be optimizations that avoid waiting for lock expiration but that's out of the range of this RFC.
- A different transaction can resolve the pessimistic lock when it encounters the pessimistic lock in `AcquirePessimisticLock` or `Prewrite`. So, if the lock is lost, `PessimisticRollback` will find no lock and do nothing. No change is needed.
- `TxnHeartBeat` will fail after the loss of pessimistic locks. But it will not affect correctness.

### Lock migration

Before lock migration, we need to scan all the memory locks in the region. To reduce unavailability time, the region will still be available to write between scanning and the region change. So the migrated locks are a stale and partial snapshot of pessimistic locks of the region. We must make sure that everything should work well after these locks are ingested.

#### Leader transfer

After `is_valid` is set to `false`, later pessimistic locks are replicated through Raft proposals. So, these new pessimistic locks won't be missing.

For the just migrated locks, we should guarantee no lock will be missing and no deleted lock will appear again on the new leader. Considering following cases with different order of operations when there is a concurrent write command that will remove an existing pessimistic lock:

1. Propose write -> propose locks -> apply write -> apply locks -> transfer leader
   Because the locks marked as deleted will not be proposed. The lock will be deleted when applying the write while not showing up again after applying the locks. On the new leader, the write command is successfully applied, so the lock information is correct.

2. Propose locks -> propose write -> transfer leader
   No lock will be lost in normal cases because the write request has been sent to the raftstore, it is likely to be proposed successfully, while the leader will need at least another round to receive the transfer leader message from the transferree.

#### Region merge

We reject all writings before proposing `PrepareMerge` and wait until the latest proposed command is applied. After that, we can make sure that no lock with a deleted mark will be deleted successfully. Either it should have been deleted because it is applied, or the write command will be rejected.

So, considering the different cases like leader transfer:

1. Propose write -> reject write -> apply write -> propose locks -> propose prepare merge
   The proposed write command will be applied successfully before proposing the existing pessimistic locks. This means the proposed locks will not include the locks that are deleted by the write command. It is correct.

2. Reject write -> propose write -> propose locks -> propose prepare merge
   The write command will be rejected and will not be applied. So, we need to propose the pessimistic locks marked as deleted, because they will not be deleted by the source region leader and should be moved to the new region.

#### Region split

Region split happens on the same node. Considering the different orders as always:

1. Propose write -> propose split -> apply write -> execute split
    The write will be applied earlier than split. So, the lock will be deleted earlier than moving locks to new regions.

2. Propose split -> propose write -> ready split -> apply write
   The write will be skipped because its version is lower than the new region. So, no lock should be deleted in this case. It is correct for us to transfer the locks with deleted marks.

3. Propose split -> ready split -> propose write
   The write proposal will be rejected because of version mismatch. So, it is correct to include the locks with deleted marks.
