# RFC: Coprocessor Cache

## Summary

This proposal introduces a cache layer between TiDB and TiKV Coprocessor so that
the performance can be greatly improved in some scenarios.

## Motivation

Sometimes data may not be changed frequently, but being queried quite often
using same SQL statements, like:

- Query a configuration table using constant predicates

- Run aggregations over fixed columns like `SELECT COUNT(1) from T`,
  `SELECT MAX(column) from T`

For those queries, the Coprocessor request send to TiKV from TiDB is likely to
be the same (except for the Timestamp), and the corresponding data at TiKV side
is likely to be unchanged. This makes it possible to cache the Coprocessor
response at region level with very minimal effort. When there are writes to the
region, the cache is invalidated. The cache invalidation happens at the region
level, so that as long as not all regions are changed, there could be useful
cache that can improve the performance.

The idea comes from https://github.com/pingcap/tidb/issues/9336.

## Detailed design

This design is very similar to the HTTP cache mechanism.

### General Flow

1. TiDB stores the cache.

2. TiDB always sends requests to TiKV.

3. TiKV responds with a tag.

4. TiDB stores the response and the tag.

5. When TiDB sends same request again to TiKV next time, the request is carried
   with the tag received from TiKV last time. The tag indicates the change of
   underlying data.

6. TiKV checks the tag and return "not modified" instead of real data if the
   tag is valid. In this case, TiKV don't need to scan and compute any data.

7. If TiDB receives "not modified", TiDB uses the cached response as the
   response.

### Detail Flow

1. On TiKV side, each Coprocessor response is carried with the `RaftApplyIndex`
   of the current region, as well as a `CanBeCached` flag.

   The increment of `RaftApplyIndex` of the same region indicates that there
   are writes to the region, which can be used to detect whether the cache
   is still effective later.

   `CanBeCached` is a flag indicating that current response can be cached. More
   specifically, when TiKV scans data and discovered a version > Ts, the
   response can not be cached. This will be explained later.

2. On TiDB side, there is a CoprocessorCache. Each cache entry holds the
   following data:

   - Response
   - Ts
   - RegionID
   - RegionRaftApplyIndex

   The cache key is the Coprocessor request data without the Ts.

   The cache is a LRU cache.

3. On TiDB side, before sending Coprocessor request, CoprocessorCache is checked
   first.

   TiDB sends the Coprocessor request with `IsCacheEnabled` and
   `RegionRaftApplyIndex` field hen the following condition is met:

   - The Request is deterministic, for example, there is no `random()` functions.
   - The Request without Ts part is already in the cache.
   - The Request is sending to the same region ID as the one in the cache.
   - The Ts to query >= the Ts in the cache.

   Notice that the scan range is encoded in the request, so that for the same
   SQL, there may be multiple different requests send to different TiKV,
   occupying different cache entry.

4. On TiKV side, if TiKV receives a Coprocessor request without `IsCacheEnabled`
   or `RegionRaftApplyIndex`, then the request is handled as usual. Otherwise,
   TiKV checks the local `RaftApplyIndex` in the snapshot before scan data.

   When two `RaftApplyIndex` are matching, it means region data is not changed
   since TiDB sent last request. Thus TiDB is carrying an up-to-date cache. TiKV
   skips the scan and skips the calculation in this case. The response will be
   marked with `IsCacheHit`.

5. On TiDB side, when receiving a response with `IsCacheHit`, TiDB uses the
   response data stored in the cache as the response data. Otherwise, TiDB
   will cache the response data using the RaftApplyIndex in the response data.

   An exception is that when the response is marked as `CanBeCached == false`,
   TiDB should not store response in the cache at all.

Since we allow requests that Query Ts > Cached Ts to utilize the cache result,
we must make sure that:

1. After Cached Ts, before Query Ts, there is no writes to the region.

   This is ensured by identical `RaftApplyIndex`.

2. There is no existing data whose Ts > Cached Ts written before Cached Ts.

   This is ensured by `CanBeCached` flag.

## Drawbacks

- This adds more memory usage to the TiDB.
- More risks.
- Different TiDB cannot share cache data.

## Alternatives

There was a previous design that cache data at TiKV side. However its
performance is not good and the implemention is very dirty as well in order to
detect changes to the region.

## Unresolved questions

None
