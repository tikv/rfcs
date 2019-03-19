# Calculated Commit TS

## Summary

Support calculating `commit_ts`es instead of allocating them from PD to reduce
latency of committing.

## Motivation

TiKV uses Percolator's transaction algorithm. When committing a transaction,
there are a few steps to do:

1. Prewrite all keys with the transaction's `start_ts`
2. Allocate the `commit_ts` from PD.
3. Commit the transaction with `commit_ts`.

The `commit_ts` allocated from PD is important in providing Snapshot Isolation,
however allocating it from PD is not the only possible way. It's possible to
calculate it instead of allocating it for every transaction. Therefore, we can
remove the latency of waiting for PD's response, as a result we can reduce
latency of committing.

## Detailed design

### How to calculate a valid `commit_ts`

To provide Snapshot Isolation, we should ensure all transactional reads are
repeatable. The `commit_ts` should be large enough so that the transaction will
not be committed before a valid read. Otherwise, Repeatable Read will be broken.
For example:

1. Txn1 gets `start_ts` 100
2. Txn2 gets `start_ts` 200
3. Txn2 reads key `"k1"` and got value `"1"`
4. Txn1 prewrites on `"k1"` with value `"2"`
5. Txn1 committed with `commit_ts` 101
6. Tnx2 reads key `"k1"` and got value `"2"`

Txn2 reads `"k1"` twice but got two different results. If the `commit_ts` is
allocated from PD, this must not happen, becuase Txn2's first read must happens
before Txn1's prewrite, and Txn1's `commit_ts` must be requested after finishing
prewrite, which means, Txn2's `commit_ts` must be larger than Txn1's `start_ts`.

In the other hand, the `commit_ts` can't be arbitrarily large. If the
`commit_ts` is ahead of the real time, the committed data may be unreadable by
other new transactions, which breaks the integrity. We are not sure whether a
timestamp is ahead of the real time if we don't ask PD.

We can get a conclusion that in order not to break the Snapshot Isolation and
the integrity, a `commit_ts` is valid in  all this range:

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

A new component named `ReadTsCache` is added to `Storage` and `Coprocessor` (and
they shares the same content). The `ReadTsCache` records `max_read_ts` of all
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

`ReadTsCache` contains a map that maps from `region_id` to `RegionMaxTsRecord`:

```rust
// Divide the map into multiple according to region_id's hash to reduce lock
// contention.
max_read_ts_map: Vec<RwLock<HashMap<u64, Arc<RwLock<RegionMaxTsRecord>>>>>,
```

Each transactional reads need to update the `max_read_ts` of the requested
region **before** getting snapshot (`u64::MAX` will be ignored). Each prewrite
need to get the `max_read_ts` after successfully writing and return it with
`PrewriteResponse`, and the TiKV client is responsible for calculating the
`commit_ts` according to the timestamps carried in `PrewriteResponse`s. If
failed to get the `max_read_ts` (due to not ready or version not match), just
returns 0 to tell the client to fall back to allocating `commit_ts` from PD.

`ReadTsCache` updates its inner region information via Raftstore observer and
collects only leader. Every time a region is added to the map or a region's
version is changed, `is_ready` will be unset until getting a timestamp from PD.

### New constraints to TiKV's txn algorithm

* If Txn1's `commit_ts` is the same as Txn2's `start_ts`, then Txn1's committed
  data is visible to Txn2
* If key `"k1"`'s latest version's `commit_ts` is the same as Txn1's `start_Ts`,
  then Txn1 prewrites on key `"k1"` will cause write conflict.
* In `CF_WRITE`, rollback record should never replace other records with the
  same key and timestamp.

### Code changes to TiKV Client

In the 2PC algorithm, if all prewrite requests returns a non-zero timestamp,
then the `commit_ts` can be calculated. Otherwise it should still request the
`commit_ts` from PD.

### Corner cases

Regions' leaders are not static. They may transfer from a TiKV to another TiKV.
Consider this case:

1. Txn1 get start_ts 50
2. Txn2 read `"k1"` with timestamp 200
3. Leader transfer (And another TiKV doesn't know the previous `max_read_ts`
   200)
4. Txn3 read `"k1"` with timestamp 100
5. Txn1 prewrite and commit on `"k1"`

If we don't carefully handle this case, Txn1's `commit_ts` may be calculated
101, then if Txn2 reads again, it will get a different value, which means, the
Repeatable Read is broken.

The similar problem also happens when merging occurs. If region1(`max_read_ts`
is 200) merges into region2(`max_read_ts` is 100), then it's difficult to know
that region2's `max_read_ts` should be updated to 200, especially if the two
regions' leaders are on different TiKVs.

That's why the `is_ready` field is here. When a region is just created in
`ReadTsCache`, or a region's version is just changed, `is_ready` will be set to
false. At the same time a background thread will get a timestamp from PD, update
the `max_read_ts` with it and set `is_ready` to true. This promises that the
`max_read_ts` is safe as far as `is_ready` is true.

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

* Possibly performance regression to transactional reads caused by lock
  contentions.
* Makes the `commit_ts` not related to the real commit time, which may make the
  data lives shorter than GC lifetime.
* Breaks historical snapshot reading.
* Increases complexity of the code.
* tidb-binlog will be much affected by this change.
* If someday we supports transactional reads on followers, it will be exclusive
  with this optimization.

It's better to made this an optional optimization, and can be enabled with
config when possible.

## Alternatives

## Unresolved questions
