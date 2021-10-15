# RFC: In-memory Pessimistic Locks

- RFC PR: https://github.com/tikv/rfcs/pull/77
- Tracking Issue: (to be filled)

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

`PeerPessimisticLocks` contains a simple `HashMap` for the memory locks. And it contains `epoch` and `valid` marking its status. `size` is used to control the used memory. They will be explained later.

```rust
pub struct PeerPessimisticLocks {
    map: HashMap<Key, PessimisticLock>,
    size: usize,
    epoch: u64,
    valid: bool,
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

Before processing an `AcquirePessimisticLock` command in the scheduler, we save the current `epoch` of `PeerPessimisticLocks`. This will be checked later when writing locks to the table.

When loading a lock, we can read the in-memory lock table first. If no lock is found, then read the underlying RocksDB.


And a new `Modify` variant is added. The `AcquirePessimisticLock` now adds `Modify::PessimisticLock` instead of a normal `Modify::Put`.

```rust
pub enum Modify {
    // ...
    PessimisticLock(Key, PessimisticLock),
}
```

Then, we introduce `ResponsePolicy::NoPropose` for this feature. If enabling this feature and `valid` is `true`, this will be the policy for `AcquirePessimisticLock` commands. Otherwise, the policy will fall back to `OnPropose`.

With this policy, we need to check the `PeerPessimisticLocks` first. If the `epoch` is different from the `epoch` before processing the command, the region must have changed and the lock table we fetched before processing is not the one we should write to. For simplicity, we just return an `EpochNotMatch` error and let the client retry. Later sections will talk about how `epoch` and `valid ` are maintained.

If the check passes, it is safe for us to write the pessimistic locks into the lock table. After this, we can return the response to the client without proposing anything at all.

For all writes involving the lock CF, the lock in the lock table should be cleared. For example, when a `Prewrite` command replaces a pessimistic lock with a 2PC lock, of course, this is a replicated write, we need to remove the pessimistic lock from the lock table after the write succeeds. This should be done in the raftstore no matter it is a leader or a follower.

### Region leader transfer

If it is a voluntary leader transfer triggered by PD, we have the chance to transfer the pessimistic locks to the new leader to avoid unexpected transaction failures.

In `pre_transfer_leader`, the leader serializes current pessimistic locks into bytes and modifies `valid` to `false`. Then, later `AcquirePessimisticLock` commands will fall back to proposing locks. In this way, we can guarantee the pessimistic locks either exist in the serialized bytes, or are replicated through Raft.

The serialized pessimistic locks are carried in the `context` of `MsgTransferLeader` sent by the leader. The context format should be designed to have forward compatibility.

By default, a Raft message has a size limit of 1 MiB. We will guarantee that the total size of in-memory pessimistic locks in a single region will not exceed the limit. This will be discussed later.

If the leader transfer is rejected, we should revert `valid` to `true`.

### Region merge

On region merge, the source region can send the pessimistic locks to the target region via `CommitMerge`.

In `on_ready_prepare_merge`, we can first set `valid` to `false` to disable writing pessimistic locks to memory. 

In `schedule_merge`, the leader of the source region can serialize current pessimistic locks into bytes and set `valid` to `false`. The serialized pessimistic locks are carried in `CommitMergeRequest`. This means we need an extra field in `CommitMergeRequest`:

```protobuf
message CommitMergeRequest {
    // ...
    bytes pessimistic_locks = 4;
}
```

Then, in `exec_commit_merge`, the target region leader deserializes the pessimistic locks from the request and passes it to the `PeerFSM` in `ExecResult::CommitMerge`. The pessimistic locks from the source region are merged to the lock table of the target region in `on_ready_commit_merge`. After it succeeds to merge regions, we increase the pessimistic locks `epoch` and set `valid` to `true`.

If the merge is rolled back, we can set `valid` of the source region back to `true`.

### Region split

Typically, after a region is split into several new regions, the leader of all new regions are still located in the same TiKV. So it will be only memory operations to handle the case.

In `on_ready_split_region`, we first set `valid` to `false`. Later pessimistic locks will either make a proposal or just fail, so we will not miss any lock.

Then, we iterate all locks and group them into new regions. After the locks are processed, we can increase `epoch` and set `valid` to `true`.

### Memory limit

To simplify handlings in region changes, we don't allow the total size of the pessimistic locks in a single region to be too large. The limit for each region is 512 KiB, matching the 1 MiB Raft message limit.

There should also be a global limit. The default size is the minimum of 1 GiB and 5% of the system memory.

It is easy for the lock writer to maintain the total size of pessimistic locks in a single region. If the memory limit is exceeded, we have to fall back to propose pessimistic locks.

### Compatibility

Pessimistic locks are sent via new fields in Raft messages. Only the upgraded TiKV instances can handle them while the old TiKV instances will ignore them. If the feature is unconditionally enabled, the pessimistic locks will be lost and the success rate of pessimistic transactions will drop during the rolling upgrade.

So this feature needs to be enabled after TiKV is fully upgraded. A possible approach is that TiKV gets the cluster version from PD and automatically turns it on through feature gate control.

The storage structure is not changed, so downgrading is feasible. However, before the downgrade, we must disable this feature first to avoid affecting the success rate of pessimistic transactions.

Ecosystem tools should not be affected by this optimization.