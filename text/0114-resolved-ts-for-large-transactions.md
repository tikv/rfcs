# Resolved-ts for Large Transactions

Author: @ekexium

Tracking issue: N/A

## Background

The RFC is a variation and extension of @zhangjinpeng87 's [Large Transactions Don't Block Watermark](https://github.com/pingcap/tiflow/blob/master/docs/design/2024-01-22-ticdc-large-txn-not-block-wm.md). They aim to solve the same problem.

Resolved-ts is a mechanism that provides a temporal guarantee. It is defined as follows: Once a resolved timestamp T is observed by any component of the system, it is guaranteed that no new commit record with a commit timestamp less than or equal to T will be subsequently produced or observed by any component of the system.

In current TiKV(v8.3), large transactions can block resolve-ts from advancing, because it is calculated as `min(pd-tso, min(lock.ts))`, which is actually a more stringent constraint than its original definition. A lock from a pipelined txn can live several hours. This will make services dependent on resolved-ts unavailable.

## Goals

- Current phase objectives:
  - Prevent pipelined transactions from impeding resolved-ts advancement.

- Long-term goal:
  - Minimize the overhead of resolved-ts maintenance.
  - Maximize resolved-ts freshness.
  - Achieve uninterrupted resolved-ts progression, addressing all potential blocking factors beyond long transactions and their associated locks. (Further details to be discussed in the final section of this proposal.)

## Design

Key Concept: Utilizing `lock.min_commit_ts` instead of `lock.start_ts` for resolved-ts calculation.

Rationale:
1. Resolved-ts definition is independent of LOCK CF.
2. Current use of `lock.start_ts` is based on the invariant: `lock.start_ts` < `lock.commit_ts`.
3. A valid resolved-ts doesn't necessitate the absence of locks with smaller start_ts, provided their future commit_ts are guaranteed to be larger.

Advantages of `lock.min_commit_ts`:
1. Satisfies a similar invariant: `lock.min_commit_ts` <= `lock.commit_ts`
2. Unlike the static `lock.start_ts`, `lock.min_commit_ts` can be dynamically increased.

### Maintanence of resolved-ts

Key objective: Maximize all TiKV nodes' awareness of large pipelined transactions during their lifetime, i.e. from their first writes to all locks being committed. These info are necessary:

1. `start_ts`
2. A fresh enough `min_commit_ts`
3. Status

#### Coordinator

For a pipelined transaction, its TTL manager is responsible for fetching a latest TSO as a candidate of min_commit_ts and update both the committer's inner state and PK in TiKV. After that, it broadcasts the `start_ts` and the new `min_commit_ts` to all TiKV stores. The PK update can be piggybacked on the `heartbeat` request.

Optionally, to avoid too many RPC overhead, the broadcast messages from different transactions can be batched.

Atomic variables or locks may be needed for synchronization between the TTL manager and the committer.

#### Scaling out TiKVs

Challenge:
During cluster expansion, when a new TiKV instance is integrated mid-transaction, the TTL manager must promptly incorporate it into its broadcast list. The TTL manager relies on the region cache for store information. However, if the region cache lacks awareness of a newly added TiKV, the TTL manager may inadvertently omit it from broadcasts.

One solution is to have an background goroutine in the region cache to periodically refresh the complete store list.

#### TiKV scheduler - heartbeat

Besides updating TTL, it also supports update min_commit_ts of the PK.

*TBD: should it also update max_ts?*

#### TiKV txn_status_cache

A standalone part was created for large transactions specially. The cache serves as

1. Provides up-to-date `min_commit_ts` information for large transactions to the resolved-ts resolver. 
1. Offers an optimized path for read requests, reducing the need to query PK for transaction status.

Cache Management Strategy:

1. Retention policy:
   - Maximize retention of useful information.
   - No eviction based on space constraints, leveraging the compact entry structure.
   - Assumption: Limited number of concurrent large transactions.

2. TTL management:
   - Implement a substantial default TTL for cache entries.
   - Rationale: Minimize redundant operations when readers encounter locks from these transactions.

3. Post-commit procedure:
   - Upon successful commitment of all secondary locks in a large transaction:
     a. Coordinator broadcasts a TTL update to all TiKV nodes.
     b. Extends TTL by several seconds.
   - Purpose: Allow follower peers time to synchronize catch up with the leader.
   - Caution: Immediate eviction may lead to stale reads encountering locks and missing the cache.

#### TiKV resolved-ts Resolver

Operational Mechanism:

1. Standard lock handling:
   - Tracks normal locks using conventional methods.

2. Large pipelined transaction Locks:
   - Identified by the "generation" field.
   - Tracks only the start_ts of locks.

Resolved-ts calculation:

- Primary Method:
  - Attempts to map start_ts to min_commit_ts via txn_status_cache.
  - Sets resolved-ts to max(min_commit_ts + 1, current_resolved_ts).
- Fallback Method:
     - Uses start_ts for calculation if cache lookup fails.



We preseve the resolved-ts semantics by ensuring that resolved-ts is always greater than or equal to min_commit_ts + 1.

When the resolver observes a LOCK DELETION event, it immediately ceases tracking the corresponding start_ts for large pipelined transactions. This action is justified because lock deletion is a clear indicator that a transaction's final state has been determined. By stopping the tracking at this point, the resolver efficiently manages its resources and maintains an up-to-date view of active transactions.

### Upgrading TiKV

This design constitutes a non-intrusive modification, eliminating specific concerns during the upgrade process. In case of a cache miss, the system automatically falls back to the original approach, ensuring seamless backward compatibility.

### Benefits in resolving locks

Across all lock resolution scenarios—including normal reads, stale reads, flashbacks, and potentially write conflicts—a preliminary txn_status_cache lookup can significantly reduce unnecessary computational overhead introduced by large transactions.

### Compatibility

The key difference is that services can now observe more locks. 

It's important to note that the current implementation still permits the encounter of locks with timestamps smaller than resolved-ts. This proposal maintains this existing behavior, thus we do not anticipate any correctness issues arising from this modification. The principal challenges we foresee are mainly performance and availability concerns.

#### Stale read

When it meets a lock, first query the txn_status_cache. When not found in the cache, fallback to leader read.

#### Flashback

1. Compatilibity with CDC: Flashback will write a lock to block resolved-ts during its execution. It does not use pipelined transaction so this lock will be treated as a normal lock.

2. The current and previous (up to v8.3) implementations of Flashback in TiKV rely on an incorrect assumption about resolved-ts guarantees. This misconception can lead to critical issues, such as the potential violation of transaction atomicity, as documented in https://github.com/tikv/tikv/issues/17415.

#### EBS snapshot backups

Its only dependency on resolved-ts is to use Flashback.

#### CDC

Already well documented in [Large Transactions Don't Block Watermark](https://github.com/pingcap/tiflow/blob/master/docs/design/2024-01-22-ticdc-large-txn-not-block-wm.md). Briefly, a refactoring work is needed.

### Cost

Memory: each cache entry takes at least 8(start_ts) + 8(min_commit_ts) + 1(status) + 8(TTL) = 33 bytes. Any TiKV instance can easily hold millions of such entries.

Latency: The additional operations required for resolved-ts maintenance can be executed asynchronously, thereby mitigating any potential impact on system latency.

RPCs: each large transaction sends N more RPCs per second, where N is the number of TiKVs. Batching can greatly reduce the RPC overhead.

CPU: the mechanism may consume more CPU, but should be ignorable.



## Possible future improvements

### Tolerate lagging non-pipelined transactions

To get closer to our ultimate goal: minimize blocking of resolved-ts, we can further consider the case where resolved-ts being blocked by normal transaction locks. Typical causes could be:

- Situation-1: Memory locks from async commit and 1PC. Normal locks are region-partitioned can will not block resolved-ts of other regions. But concurrenty manager is a node-level instance. Memory locks can block every (leader) region in the same TiKV.
- Situation-2: Slow transactions which take too much time committing their locks
- Situatino-3: Long-running transactions that may not be large.
- Situation-4: Node failures, network jitters, etc.

#### Approach-1: resolver pushing min_commit_ts

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

#### Approach-2: long-running transactions setting min_commit_ts

If a transaction already runs for a long time, it must get a latest TSO as its min_commit_ts before starts prewriting, if it's not using async commit or 1PC. This prevents the short-lived locks blocking resolved-ts, whether they are memory locks or not.
