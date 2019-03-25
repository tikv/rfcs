# Calculated Commit TS

## Summary

This RFC proposes a feature of calculating `commit_ts` instead of allocating
`commit_ts` from PD to reduce latency of committing.

## Motivation

TiKV uses Percolator's transaction algorithm. When committing a transaction,
there are a few steps to do:

1. Prewrite all keys with the transaction's `start_ts`
2. Allocate the `commit_ts` from PD.
3. Commit the transaction with `commit_ts`.

The `commit_ts` allocated from PD is important in providing Snapshot Isolation,
however allocating it from PD is not the only possible way. It's possible to
calculate it instead of allocating it for every transaction. Therefore, we can
remove the latency of waiting for PD's response, thereby reducing commit
latency.

## Detailed design

### How to calculate a valid `commit_ts`

To provide Snapshot Isolation, we should ensure all transactional reads are
repeatable. The `commit_ts` should be large enough so that the transaction will
not be committed before a valid read. Otherwise, Repeatable Read will be broken.
For example:

1. Txn1 gets `start_ts` 100
2. Txn2 gets `start_ts` 200
3. Txn2 reads key `"k1"` and gets value `"1"`
4. Txn1 prewrites `"k1"` with value `"2"`
5. Txn1 commits with `commit_ts` 101
6. Tnx2 reads key `"k1"` and gets value `"2"`

Txn2 reads `"k1"` twice but gets two different results. If `commit_ts` is
allocated from PD, this will not happen, because Txn2's first read must happen
before Txn1's prewrite while Txn1's `commit_ts` must be requested after
finishing prewrite. And as a result, Txn2's `commit_ts` must be larger than
Txn1's `start_ts`.

On the other hand, `commit_ts` can't be arbitrarily large. If the `commit_ts` is
ahead of actual time, the committed data may be unreadable by other new
transactions, which breaks the integrity. We are not sure whether a timestamp is
ahead of the real time if we don't ask PD.

To conclude, in order not to break the Snapshot Isolation and the integrity, a
valid range for `commit_ts` should be:

```text
max{start_ts, max_read_ts_of_written_keys} < commit_ts <= now
```

So here comes an idea that we can calculate the commit_ts this way:

```text
commit_ts = max{start_ts, region_1_max_read_ts, region_2_max_read_ts, ...} + 1
```

where `region_N_max_read_ts` is the maximum timestamp of all reads on the
region, for all regions involved in the transaction.

### Code changes to TiKV

A new component named `ReadTsCache` will be added to `Storage` and `Coprocessor`
(and they share the same content). `ReadTsCache` records `max_read_ts` of all
leaders on this store. For each leader, these fields are saved:

```rust
struct RegionMaxTsRecord {
    /// The `max_read_ts` is not safe to use immediately after creation or
    /// updating of the region. When the `max_read_ts` is ready, `is_ready` will
    /// be set.
    is_ready: bool,

    /// The version of the region's epoch. The `max_read_ts` can only be used
    /// when `version` matches the prewrite request's version.
    version: u64,

    /// The maximum timestamp of transactional reads on this region. Note that
    /// in order to ensure the `max_read_ts` is no less than reads on the region
    /// before the region's leader goes to the current store, A timestamp will
    /// be fetched from PD after creating or updating of the region.
    max_read_ts: u64,
}
```

`ReadTsCache` contains mapping from `region_id` to `RegionMaxTsRecord`:

```rust
// Divide the map into multiple according to region_id's hash to reduce lock
// contention.
max_read_ts_map: Vec<RwLock<HashMap<u64, Arc<RwLock<RegionMaxTsRecord>>>>>,
```

Each transactional read needs to update the `max_read_ts` of the requested
region **before** getting snapshot (`u64::MAX` will be ignored). Each prewrite
needs to get the `max_read_ts` after writing successfully and return it with
`PrewriteResponse`, and the TiKV client is responsible for calculating the
`commit_ts` according to the timestamps carried in `PrewriteResponse`s. If TiKV
can't provide `max_read_ts` for a Region (due to not ready or version mismatch),
just returns 0 to tell the client to fall back to allocating `commit_ts` from
PD.

`ReadTsCache` updates its inner region information via Raftstore observer and
only collects leader information. Each time a region is added to the map or a
region's version is changed, `is_ready` will be unset until getting a timestamp
from PD.

### New constraints to TiKV's txn algorithm

* If Txn1's `commit_ts` is the same as Txn2's `start_ts`, then Txn1's committed
  data is visible to Txn2
* If `commit_ts` of key `"k1"`'s latest version is the same as Txn1's
  `start_Ts`, then Txn1's prewrites on key `"k1"` will cause a write conflict.
* In `CF_WRITE`, a rollback record should never replace other records with the
  same key and timestamp.

### Code changes to TiKV Client

In the 2PC algorithm, if all prewrite requests return a non-zero timestamp,
then `commit_ts` can be calculated. Otherwise, it still needs to request the
`commit_ts` from PD.

### Corner cases

Regions' leaders are not static. They may transfer from one TiKV instance to
another. Consider this case:

1. Txn1 gets `start_ts` 50
2. Txn2 reads `"k1"` with timestamp 200
3. Leader transfers to another TiKV that doesn't know the previous `max_read_ts`
   200
4. Txn3 reads `"k1"` with timestamp 100
5. Txn1 prewrites and commits `"k1"`

If we don't handle this case carefully, Txn1's `commit_ts` may be calculated as
101, then if Txn2 reads again, it will get a different value, which means,
Repeatable Read is broken.

The similar problem may also happen for merging. If region1 (`max_read_ts` is
200) merges into region2 (`max_read_ts` is 100), it's difficult to know that
region2's `max_read_ts` should be updated to 200, especially when the leaders
of the two regions are on different TiKVs.

That's why the `is_ready` field is here. When a region is just created in
`ReadTsCache`, or a region's version is just changed, `is_ready` will be set to
false. At the same time a background thread will get a timestamp from PD, update
the `max_read_ts` with it and set `is_ready` to true. This guarantees that
`max_read_ts` is safe as long as `is_ready` is true.

Another problem is that, now it's possible that a transaction's `commit_ts` is
the same as another transaction's `start_ts`. In the current implementation in
TiKV, a committed transaction will leave a commit record with key
`key{commit_ts}` in `CF_WRITE`, and a committed transaction will leave a
rollback record `key{start_ts}` in `CF_WRITE`. So there may be key collision
in `CF_WRITE`. The solution is, when writing Rollback record to `CF_WRITE`, if
there's already another record with the same key and timestamp, then the
Rollback will not be written; when a prewrite finds there's a record in
`CF_WRITE` with the same key and timestamp, the prewrite will fail.

## Drawbacks

* Possible performance regression to transactional reads caused by lock
  contentions.
* Detaches `commit_ts` from the actual commit time, which may make the data
  lives shorter than GC lifetime.
* Breaks historical reads.
* Increases complexity of the code.
* tidb-binlog will be much affected by this change.
* If a user reads TiKV with a timestamp that is ahead of actual time (probably
  caused by calling TiKV client directly), a transaction might be committed with
  a `commit_ts` that is ahead of the real time.

It's better to make this an optional optimization that can be enabled with
config when it's needed.

## Alternatives

## Unresolved questions

The performance impacts to reads with shuffle leader enabled still needs to be
carefully dealt with. When a leader transfers, the outer lock of
`max_read_ts_map` need to be write locked. Therefore, other read/write
operations, which only need read lock, may be affected. According to tests, this
may make QPS of read only queries 3% lower (the tested queries are like
`SELECT v FROM table WHERE id = a OR id = b`). Luckily, frequent leader transfer
is not a common case.
