# Resolved-ts for Large Transactions

Author: @ekexium

Tracking issue: https://github.com/tikv/tikv/issues/17459

## 1. Background

This RFC is a variation and extension of @zhangjinpeng87's [Large Transactions Don't Block Watermark](https://github.com/pingcap/tiflow/blob/master/docs/design/2024-01-22-ticdc-large-txn-not-block-wm.md). They aim to solve the same problem.

Resolved-ts is a mechanism that provides a temporal guarantee. It is defined as follows: Once a resolved-ts T is observed by any component of the system, it is guaranteed that no new commit record with a commit-ts less than or equal to T will be subsequently produced or observed by any component of the system.

In current TiKV(v8.3), large transactions can block resolve-ts from advancing, because it is calculated as `min(pd-tso, min(lock.ts))`, which is actually a more stringent constraint than its original definition. A lock from a pipelined txn can live several hours. This will make services dependent on resolved-ts unavailable.

## 2. Goals

- Current phase objectives:
  - Prevent pipelined transactions from impeding resolved-ts advancement.

- Long-term goal:
  - Minimize the overhead of resolved-ts maintenance.
  - Maximize resolved-ts freshness.
  - Achieve uninterrupted resolved-ts progression, addressing all potential blocking factors beyond large transactions.

## 3. Proposed Design

### 3.1 Key Idea

**Proposed Change:** Utilize `lock.min_commit_ts` instead of `lock.start_ts` for resolved-ts calculation.

Rationale:
1. Resolved-ts definition is independent of LOCK CF.
2. Current use of `lock.start_ts` is based on the invariant: `lock.start_ts` < `lock.commit_ts`.
3. A valid resolved-ts doesn't necessitate the absence of locks with smaller start_ts, as long as their future commit_ts are guaranteed to be larger.

Advantages of `lock.min_commit_ts`:
1. Satisfies a similar invariant: `lock.min_commit_ts` <= its `commit_ts`
2. Unlike the constant `lock.start_ts`, `lock.min_commit_ts` can be dynamically increased.

### 3.2 Maintenance of resolved-ts

Key objective: Maximize all TiKV nodes' awareness of large pipelined transactions during their lifetime.

#### 3.2.1 Coordinator

**Proposed Change:** 
- The TTL manager goroutine fetches latest TSO as min_commit_ts candidate.
- Updates committer's inner state and PK in TiKV.
- `min_commit_ts` updates are piggybacked on `heartbeat` request.
- Broadcasts `start_ts` and new `min_commit_ts` to all TiKV stores. A new type of message will be introduced in kvproto.
- Optional: Batch broadcast messages to reduce RPC overhead.

Atomic variables or locks may be needed for synchronization between TTL manager and committer.

#### 3.2.2 Scaling out TiKVs

**Challenge:** New TiKV instances integrated mid-transaction may be omitted from broadcasts if region cache is unaware.

**Proposed Solution:** Implement a background goroutine in the region cache to periodically refresh the full store list.

An alternative solution is to let PD push the full store list to TiDBs when the cluster topology changes. However, there is no existing mechanism for this, and we don't want to introduce a new one, as it may be too complex.

#### 3.2.3 TiKV scheduler - heartbeat

**Proposed Change:** Extend heartbeat functionality to support updating min_commit_ts of the PK.

*TBD: Consider whether it should also update max_ts.*

#### 3.2.4 TiKV txn_status_cache

**Proposed Change:** Introduce a standalone part next to the existing cache for large transactions that:
1. Provides `min_commit_ts` information for resolved-ts resolver.
2. Offers optimized path for read requests, reducing PK queries for transaction status.

Given the condition that given the condition that the number of concurrent pipelined transactions is limited, we want the cache to satisfy the following requirements:
  - Maximize retention of useful information.
  - Entries should be discarded based on coordinator directives, not due to TTL expiration or cache capacity limits.
  - Maximize hit rate, i.e. the eviction of an transaction entry should happen after lock release, especially for follower peers.

**Proposed Cache Implementations:**
1. A large enough capacity to minimize full cache evictions. 
2. Compact cache entries to minimize memory overhead.
3. A large enough default TTL.
4. Post-commit procedure after committing all secondary locks:
   a. Coordinator broadcasts TTL update to all TiKV nodes.
   b. Extends TTL by several seconds.

#### 3.2.5 TiKV resolved-ts Resolver

**Proposed Operational Mechanism:**
1. Standard lock handling remains unchanged.
2. For large pipelined transaction locks:
   - Identify by "generation" field.
   - Track only `start_ts` of locks.
3. Stop tracking `start_ts` for large pipelined transactions upon LOCK DELETION event.

**Proposed Resolved-ts calculation:**
- Primary Method:
  - Map `start_ts` to `min_commit_ts` via txn_status_cache.
  - Set resolved-ts to `max(min_commit_ts + 1, current_resolved_ts)`.
- Fallback Method:
  - Use start_ts if cache lookup fails.

### 3.3 Upgrading TiKV

This proposal is non-intrusive, with automatic fallback to original approach on cache miss, ensuring backward compatibility.

### 3.4 Benefits in resolving locks

Preliminary txn_status_cache lookup can reduce unnecessary overhead for various lock resolution scenarios.

## 4. Compatibility Considerations

Key difference: Services may observe more locks.

**Note:** Current implementation still permits the encounter of locks with timestamps smaller than resolved-ts. This proposal maintains this existing behavior, thus we do not anticipate any correctness issues arising from this modification. The principal challenges we foresee are mainly performance and availability concerns.

### 4.1 Stale read

**Proposed Approach:** Query txn_status_cache first when encountering a lock. Fallback to leader read if not found in cache.

### 4.2 Flashback

1. Compatibility with CDC: Flashback will write a lock to block resolved-ts during its execution. It does not use pipelined transaction so the behavior is not affected.
2. The current and previous (up to v8.3) implementations of Flashback in TiKV rely on an incorrect assumption about resolved-ts guarantees. This misconception can lead to critical issues, such as the potential violation of transaction atomicity, as documented in https://github.com/tikv/tikv/issues/17415.

### 4.3 EBS snapshot backups

No direct impact. Only dependency on resolved-ts is through Flashback.

### 4.4 CDC

Refactoring work needed, as documented in [Large Transactions Don't Block Watermark](https://github.com/pingcap/tiflow/blob/master/docs/design/2024-01-22-ticdc-large-txn-not-block-wm.md).

## 5. Cost Analysis

- Memory: the minimum memory for each entry is 8(start_ts) + 8(min_commit_ts) + 1(status) + 8(TTL) = 33 bytes. TiKV instances can hold millions of entries.
- Latency: Minimal impact due to asynchronous execution of additional operations.
- RPCs: Increased RPC count, could be mitigated by batching.
- CPU: Slight increase, expected to be negligible.

## 6. Possible future Improvements

### 6.1 Tolerate lagging non-pipelined transactions

To further minimize resolved-ts blocking, consider addressing:
1. Memory locks from async commit and 1PC
2. Slow transactions which take too much time committing their locks
3. Long-running transactions (not necessarily large)
4. Node failures, network issues, etc

#### 6.1.1 Approach-1: resolver pushing min_commit_ts

Concept: Enable the resolver to push `min_commit_ts` for normal transactions, similar to large transactions.

However, locks using async commit cannot be pushed.

To manage overhead, cap the number of transactions that can be pushed per region based on their `min_commit_ts`.

#### 6.1.2 Approach-2: long-running transactions setting min_commit_ts

Proposal: Long-running transactions (non-async commit, non-1PC) must obtain latest TSO as min_commit_ts before prewriting.

Benefit: Prevents short-lived locks from blocking resolved-ts, including memory locks.