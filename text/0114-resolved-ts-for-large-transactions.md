# Resolved-ts for Large Transactions

Author: @ekexium

Tracking issue: N/A

## Background

The RFC is a variation of @zhangjinpeng87 's [Large Transactions Don't Block Watermark](https://github.com/pingcap/tiflow/blob/master/docs/design/2024-01-22-ticdc-large-txn-not-block-wm.md). They aim to solve the same problem.

Resolved-ts is a tool for other services. It's definition is that no new commit records smaller than the resolved-ts will be observed after you observe the resolved-ts.

In current TiKV(v8.3), large transactions can block resolve-ts from advancing, because it is calculated as `min(pd-tso, min(lock.ts))`, which is actually a more stringent constraint than its original definition. A lock from a pipelined txn can live several hours. This will make services dependent on resolved-ts unavailable.

## Goals

In current phase, our primary goal is to not let **large pipelined transactions** block the advance of resolved-ts. We focus on large pipelined transactions here. It could be adapted for general "large" transactions.

Our ultimate goal is to achieve an unblocked resolved-ts progression. Besides long transactions and their locks, there are other factors that can block the advance of resolved-ts. We will discuss it in the last part of the proposal.

## Assumptions

We assume that the number of concurrent pipelined transactions is bounded, not exceeding 10000, for example. 

This constraint is not a strict limit, but rather serves to manage resource utilization and facilitate performance evaluation. 10000 should be large enough in real world.

## Design

The key idea is using `lock.min_commit_ts` to calculate resolved-ts instead of `lock.start_ts`.

A resolved timestamp (resolved-ts) ensures that all historical events before this point are finalized and observable. In this context, 'historical events' specifically mean write and rollback records, excluding locks in the LOCK CF. Importantly, a valid resolved-ts doesn't require the absence of earlier locks, as long as their transactions' status is determined.

### Maintanence of resolved-ts

Key objective: Maximize all TiKV nodes' awareness of large pipelined transactions during their lifetime, i.e. from their first writes to all locks being committed. These info are necessary:

1. start_ts
2. Recent min_commit_ts
3. Status

#### Coordinator

For a large pipelined transaction, its TTL manager is responsible for fetching a latest TSO as a candidate of min_commit_ts and update both the committer's inner state and PK in TiKV. After that, it broadcast the start_ts and the new min_commit_ts to all TiKV stores. The update of PK can be done within the heartbeat request.

Atomic variables or locks may be needed for synchronization between the TTL manager and the committer.

#### Scaling out TiKVs

When a new TiKV instance is added to the cluster in the middle of a large transaction, its TTL manager must broadcast to it in time. TTL manager gets the list of stores from the region cache. If region cache is unaware of any newly up TiKV, TTL manager may miss it.

To mitigate this, we propose implementing an optional routine in the region cache to periodically fetch all stores.

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

For locks in large pipelined transactions, the resolver only tracks the start_ts. When calculating resolved-ts, it first attempts to map start_ts to min_commit_ts via the txn_status_cache. To maintain semantics, resolved-ts must be at least min_commit_ts + 1. If the cache lookup fails, it falls back to using start_ts for calculation.

Upon observing a LOCK DELETION, the resolver ceases tracking the corresponding start_ts for large pipelined transactions. This is justified as lock deletion only occurs once a transaction's final state is determined.

### Benefits in resolving locks

Across all lock resolution scenarios—including normal reads, stale reads, flashbacks, and potentially write conflicts—a preliminary txn_status_cache lookup can significantly reduce unnecessary computational overhead introduced by large transactions.

### Compatibility

The key difference is that services can now observe much more locks. 

Note that the current implementation still allows encountering locks with timestamps smaller than the resolved timestamp. This proposal doesn't change this behavior, so we don't anticipate correctness issues with this change. The main challenges will be related to performance and availability.

#### Stale read

When it meets a lock, first query the txn_status_cache. When not found in the cache, fallback to leader read.

#### Flashback

1. Compatilibity with CDC: Flashback will write a lock to block resolved-ts during its execution. It does not use pipelined transaction so this lock will be treated as a normal lock.

2. The current and previous (up to v8.3) implementations of Flashback in TiKV rely on an incorrect assumption about resolved-ts guarantees. This misconception can lead to critical issues, such as the potential violation of transaction atomicity, as documented in https://github.com/tikv/tikv/issues/17415.

#### EBS snapshot backups

*TBD*

It depends on Flashback.

#### CDC

Already well documented in [Large Transactions Don't Block Watermark](https://github.com/pingcap/tiflow/blob/master/docs/design/2024-01-22-ticdc-large-txn-not-block-wm.md). Briefly, a refactoring work is needed.

### Cost

Memory: each cache entry takes at least 8(start_ts) + 8(min_commit_ts) + 1(status) + 8(TTL) = 33 bytes. Any TiKV instance can easily hold millions of such entries.

Latency: maintenance of resolved-ts requires extra work, but they can be asynchoronous, thus not affecting latency.

RPCs: each large transaction sends N more RPCs per second, where N is the number of TiKVs.

CPU: the mechanism may consume more CPU, but should be ignorable.



## Possible future improvements

#### Tolerate lagging non-pipelined transactions

To get closer to our ultimate goal: minimize blocking of resolved-ts, we can further consider the case where resolved-ts being blocked by normal transaction locks. Typical causes could be:

- Memory locks from async commit and 1PC. Normal locks are region-partitioned can will not block resolved-ts of other regions. But concurrenty manager is a node-level instance. Memory locks can block every (leader) region in the same TiKV.
- Slow transactions which take too much time committing their locks
- Long-running transactions that may not be large.
- Node failures



Resolved-ts must continuously progress. However, it can't advance autonomously while ignoring locks. Such advancement would require the commit PK operation to either complete before the resolved-ts reaches a certain point or fail. This guarantee is not feasible.

The left approach feasible to prevent resolved-ts blocked by normal transactions are actively pushing their min_commit_ts, similar to what is done to large transactions. 

However, locks using async commit cannot be pushed.

To sum up, when a resolver meets a lock whose min_commit_ts still blocks its 

- Check the cache
  - Found if T.min_commit_ts >= R_TS candidate -> skip the lock
  - Else, fallthrough

- 2PC locks, check_txn_status and try to push its min_commit_ts.
  - Committed -> return its commit_ts
    - Commit_ts > R_TS candidate -> skip the lock
    - Commit_ts < R_TS candidate -> block at commit_ts - 1.
  - Min commit ts pushed, or min_commit_ts > R_TS candidate -> skip the lock
  - Rolled back -> skip the lock
  - Else -> block at min_commit_ts - 1
- Async commit locks -> check its status
  - Committed, same as 2PC locks
  - Rolled back -> skip the lock
  - Else if min_commit_ts > R_TS candidate -> skip the lock
  - Else -> block at min_commit_ts + 1

Locks belonging to the same transaction can be consolidated.

To mitigate uncontrollable overhead and metastability risks, we limit our check to K transactions per region with the lowest min_commit_ts values. This approach is necessary given the potentially substantial total number of transactions.

#### Reduce write-read conflicts

Read requests typically require a check_txn_status to advance the min_commit_ts. We propose allowing large transactions to set their min_commit_ts to a higher value, potentially exceeding the current TSO. These min_commit_ts values, stored in the txn_status_cache, would enable read requests encountering locks to bypass them via a cache lookup. Large transactions would cease this special min_commit_ts setting once ready for prewrite.
