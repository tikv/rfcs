# Resolved-ts for Large Transactions

Author: @ekexium

Tracking issue: N/A

## Background

The RFC is a variation of @zhangjinpeng87 's [Large Transactions Don't Block Watermark](https://github.com/pingcap/tiflow/blob/master/docs/design/2024-01-22-ticdc-large-txn-not-block-wm.md). They aim to solve the same problem.

Resolved-ts is a tool for other services. It's definition is that no new commit records smaller than the resolved-ts will be observed after you observe the resolved-ts.

In current TiKV(v8.3), large transactions can block resolve-ts from advancing, because it is calculated as `min(pd-tso, min(lock.ts))`, which is actually a more stringent constraint than its original definition. A lock from a pipelined txn can live several hours. This will make services dependent on resolved-ts unavailable.

## Goals

Do not let **large pipelined transactions** block the advance of resolved-ts.

We focus on large pipelined transactions here. It could be adapted for general "large" transactions.

## Assumptions

We assume that the number of concurrent pipelined transactions is bounded, not exceeding 10000, for example. 

This constraint is not a strict limit, but rather serves to manage resource utilization and facilitate performance evaluation. 10000 should be large enough in real world.

## Design

The key idea is using `lock.min_commit_ts` to calculate resolved-ts instead of `lock.start_ts`.

A resolved-ts guarantees that all historical events prior to this timestamp are finalized and observable. 'Historical events' in this context refer specifically to write records and rollback records, but explicitly exclude locks. It's important to note that the absence of locks with earlier timestamps is not a requirement for a valid resolved-ts, as long as the status of their corresponding transactions is definitively determined.

### Maintanence of resolved-ts

Key objective: Maximize TiKV nodes' awareness of large pipelined transactions, including:

1. start_ts
2. Recent min_commit_ts
3. Status

#### Coordinator

For a large pipelined transaction, its TTL manager is responsible for fetching a latest TSO as a candidate of min_commit_ts and update both the committer's inner state and PK in TiKV. After that, it broadcast the start_ts and the new min_commit_ts to all TiKV stores. The update of PK can be done within the heartbeat request.

Atomic variables or locks may be needed for synchronization between the TTL manager and the committer.

#### TiKV scheduler - heartbeat

Besides updating TTL, it can also update min_commit_ts of the PK.

*TBD: should it also update max_ts?*

#### TiKV txn_status_cache

A standalone part was created for large transactions specially. The cache serves as

1. A fresh enough source of min_commit_ts info of large transactions for resolved-ts resolver
2. A fast path for read requests when they would otherwise return to coordinator to check PK's min_commit_ts.

##### Eviction

We would keep as much useful info as possible in the cache, and never evict any of them because of space issue. One entry only contains information like start_ts + min_commit_ts + status + TTL, which should make the cache small enough, considering our assumption of the number of ongoing large transactions.

There should be a large defaut TTL of these entries, because we want to save unnecessary efforts when some reader meets a lock belonging to these transactions.

After the successfully commiting all secondary locks of a large transaction, the coordinator explicitly broadcasts a TTL update to all TiKV nodes, extending it to several seconds later. Don't immediately evict the entry to give the follower peers some time to catch up with leader, otherwise a stale read may encounter a lock and miss the cache.

#### TiKV resolved-ts resolver

Resolver tracks normal locks as usual, but handles locks belonging to large pipelined transactions in a different way. The locks can be identified via the "generation" field.

For a lock belonging to a large pipelined transaction, the resolve only tracks its start_ts. When calculating resolved-ts, the resolver first tries to map the start_ts to its min_commit_ts by querying the txn_status_cache. If not found in cache, fallback to calculate using start_ts.

Upon observing a LOCK DELETION, the resolver ceases tracking the corresponding start_ts for large pipelined transactions. This is justified as lock deletion only occurs once a transaction's final state is determined.

### Compatibility

The key difference is that services can now observe locks. They need to handle the locks.

#### Stale read

When it meets a lock, first query the txn_status_cache. When not found in the cache, fallback to leader read.

#### Flashback

*TBD*

#### EBS snapshot backups

*TBD*

#### CDC

Already well documented in [Large Transactions Don't Block Watermark](https://github.com/pingcap/tiflow/blob/master/docs/design/2024-01-22-ticdc-large-txn-not-block-wm.md). Briefly, a refactoring work is needed.

### Cost

Memory: each cache entry takes at least 8(start_ts) + 8(min_commit_ts) + 1(status) + 8(TTL) = 33 bytes. Any TiKV instance can easily hold millions of such entries.

Latency: maintenance of resolved-ts requires extra work, but they can be asynchoronous, thus not affecting latency.

RPCs: each large transaction sends N more RPCs per second, where N is the number of TiKVs.

CPU: the mechanism may consume more CPU, but should be ignorable.

