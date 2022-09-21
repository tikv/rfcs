# Enhanced Pessimistic Lock Queueing

- RFC PR: https://github.com/tikv/rfcs/pull/100
- Tracking Issue: https://github.com/tikv/tikv/issues/13298

## Summary

Provide an enhanced lock waiting & waking up behavior for pessimistic lock requests that involves only one single key. In workloads where there are frequent lock conflicts, this is expected to be able to reduce the tail latency of transactions due to unstable queueing order and useless statement-level retrying from TiDB.

## Motivation

Currently, we are adopting a simple but rough implementation for pessimistic lock waiting and waking up. It works just fine in most normal cases, however, there will be problems under workloads that has frequent lock conflicts. When there are many requests trying to acquire the lock on a single key in TiKV:

- The order that the transactions successfully get the lock has too much randomness
- When a pessimistic lock request is woken up after waiting a while in TiKV, it always returns `WriteConflict` to the client (usually TiDB) to notify the client to retry the current operation (usually a SQL statement). During retrying, it's possible that the lock is acquired by another transaction.
- When many pessimistic lock requests are waiting on the same key in TiKV and the current lock is released, one of the requests will be woken up, and the others will be woken up too after delaying for `wake-up-delay-duration` (which is a configurable value that defaults to 10ms). If the first awakened transaction's retrying request arrives late, the key might already have been locked by another transaction.

This may lead to some problems:

- The latency to acquire the lock might be very unstable, causing high tail latency.
- There might be too many statement-retries that are useless but waste resources.
- When the problem is severe, it's also possible to cause errors like lock wait timeout and statement retry limit exceeded.

By carefully adjusting `wake-up-delay-duration`, we are actually able to try to make it work well in most scenarios. However, it's very hard to adjust it since it's difficult to predict the effect of changing it. It's even harder if a user who doesn't understand the internal implementation of TiKV want to tune it. Besides, its an instance level config, which means, if a user have more than one workloads that has different characteristics in a single cluster, it's impossible to use different `wake-up-delay-duration` for different workloads.

We expects it would be better to make transactions in such kind of scenarios to execute sequentially, avoiding too much useless statement retries and high tail latency. However, this will be very complicated to implement in our current architecture, especially considering that we needs to guarantee the backward compatibility. We have a complicated design for this problem and it took us much efforts to trying to demonstrate it. We've confirmed that the complicated design is helpful for solving the problem stated above in those kinds of scenarios. However, the stronger queueing behavior in lock waiting introduces higher average latency, or to say, lower QPS. We do have ideas to optimize it further, however, it would make the design even more complicated.

Now we are going to productize the idea. As the first step, we will first implement a subset of the original design: we will enhance the queueing behavior for pessimistic lock requests that involves only one key. This will be helpful for many cases that has frequent lock conflicts on some single rows. And since there's possibly QPS regression, the new behavior will be optional and disabled by default. A user can choose to get lower tail latency at the expense of higher average latency if the user needs.

## Detailed design

The basic idea of the optimization is: when a pessimistic lock request is woken up after being blocked by another lock, try to grant it the lock immediately, instead of returning to the client and let the client retry. By this way, we avoids a transaction being woken up failed to get the lock due to the lock being preempted by another transaction.

Therefore, the essential points of the optimization are:

- Allow acquiring pessimistic locks even when there are write conflicts (the `commit_ts` of the latest version is greater than `for_update_ts` of the current request).
- When a lock is released and a queueing pessimistic lock request is woken up, allow the latter request to resume and continue acquiring the lock (it's very likely to succeed since we allow locking with conflict).
- If many pessimistic lock requests are waiting for the same key, after waking up one of them, the others don't need to be all woken up.

These new mechanisms is optional and is only available for pessimistic requests that involves only one key. Requests that choose to use or not to use the new mechanisms should be able to coexist.

### Protocol

A new field will be added to `PessimisticLockRequest` to indicate whether the new behavior is enabled in the current request:

```diff
+ enum PessimisticLockWaitingMode {
+     Legacy = 0;
+     LockAfterWokenUp = 1;
+ }

  enum PessimisticLockRequest {
      // ...
+     PessimisticLockWaitingMode lock_waiting_mode = x;
  }
```

In the new `LockAfterWokenUp` mode, the pessimistic lock request will be allowed to lock when there is write conflict, but it need to notify the client that there is such conflict. Therefore following change will be made on the response:

```diff
+ enum PessimisticLockKeyResultType {
+     Empty = 0;
+     Value = 1;
+     Existence = 2;
+     LockedWithConflict = 3;
+     Failed = 4;
+ }

+ message PessimisticLockKeyResult {
+     PessimisticLockKeyResultType type = 1;
+     bytes value = 2;
+     bool existence = 3;
+     uint64 locked_with_conflict_ts = 4;
+ }

  message PessimisticLockResponse {
      // ...
+     repeated PessimisticLockKeyResult results = x;
  }
```

The `results` field is repeated since we may expand the optimization to support multiple-keys requests too.

### Lock Waiting Queue

We will need a new waiting queue for each key. The original `WaiterManager` doesn't meet our need well, because the following reasons:

- When the current command (maybe a commit or rollback) releases a lock and wakes up another acquire_pessimistic_lock command, we expect the latter command can success. To avoid another request arrives between the two command's execution and preempts the lock, we need to know on which key we are going to wake up a lock-waiting request, do not release its latch, and let the resumed command derive the latch instead of re-acquiring it.
- We need to find blocked requests by key instead of by key hash, to avoid waking up wrong waiters and cause invalid execution of the command.
- When waking up a lock-waiting request, we need the full parameters of that request and schedule a new command in scheduler to execute it.

We are going to design a new lock waiting queue to meet our new requirement, as well as provide more extendability and possibility for future optimizations.

The lock waiting queue can be accessed in the following places:

- Pessimistic lock request encounters another lock: an entry should be enqueued to the waiting queue before the pessimistic lock command exits scheduler.
- Releasing a lock (i.e. committing or rolling back a lock): it needs to check if there's any pessimistic lock request being blocked on this lock, wake one of them up if necessary. For requests that don't enable the new mechanism, the waking up procedure should be done before calling `engine.async_write` in `Scheduler::process_write` to return the response to client as early as possible. In this way, we can prevent the performance of the old locking mode from regressing.
- A lock is successfully acquired by someone. It's possible that the state of a key transits from not-locked into locked while there are many pessimistic lock requests blocked on the key. For example, a transaction releases the lock and then one of the queueing transactions acquires the lock. In this case, the other queueing transactions is waiting for a different transaction from before, and the waiting information of the queueing transactions need to be updated in order to provide correct diagnostic information about lock waiting and do correct deadlock detection.
- Delayed notify-all for requests that uses the old behavior. As we mentioned before, in the original version, after waking up one of the queueing requests, the other queueing requests on the same key will be woken up too after `wake-up-delay-duration`. This behavior should be kept in our new implementation for requests that choose not to use the new mode.

Each queue will be a priority queue that pops the item with minimal `start_ts`, implemented by the std `BinaryHeap`. It's possible to extend the way to calculate the priority in the future, for example, considering its total wait time or the amount of transactions it's blocking.

We put the queues in a concurrent hashmap indexed by key, and put the hashmap inner the `Scheduler`.

#### Lazy-cleaning up

As we will explain later, when a request is waiting in queue, it's possible that it exceeds its timeout, or encounters deadlock. `WaiterManager` will be responsible to cancel it. However, the corresponding entry in the queue won't be removed from the priority queue efficiently. Therefore, we can use a lazy-cleaning up approach: when popping an entry from a queue, if the entry is cancelled but it is canceled (which can be indicated by an atomic flag), drop it and continue popping the next.

But this is not a complete solution: it's possible that some stale entries may be left in the queues. For example, if the leader of a region is transferred when there are some lock-waiting requests, then the waking up operation will be performed in the new leader, therefore the entries in the queue will wait until timeout without being cleaned up.

Some possible solutions:

- Use an atomic counter to count valid entries in the queue of each key. When it becomes 0 but the queue is not empty, remove it from the map.
- Periodically iterate the map and remove the stale entries.
- Introduce a priority queue that supports efficiently removing (by either introducing third-party library or implementing by ourselves), and remove the entry when it's cancelled outside.

### Waiter manager and deadlock detector adaption

`WaiterManager` no longer have the responsibility to wake up lock requests after the lock is released. Waking up will be handled in scheduler by accessing the waiting queues directly. When waking up a request, the corresponding waiter in `WaiterManager` should be removed. In order to correctly find the waiter, we will need it to be able to index waiters by unique tokens. Therefore both its interface and inner data structure need to be updated.

Considering compatibility, there will be some more troublesome details. Of course we expect that the deadlock detector can use the unique token to identify requests too, however, during rolling upgrading, a deadlock detection may be performed remotely on an old TiKV instance. We have no choice but keeping the ability to index waiters by the key hash. The inner data structure would be like:

```rust
struct WaitTable {
    // Map lock hash and ts to waiters.
    // For compatibility.
    wait_table: HashMap<(u64, TimeStamp), LockWaitToken>,
    // Map region id to waiters belonging to this region, along with other meta of the region.
    region_waiters: HashMap<u64, (RegionState, HashSet<LockWaitToken>)>,
    waiter_pool: HashMap<LockWaitToken, Waiter>,
    // ...
}
```

There's an additional map named `region_waiter` which provides the ability to cancel all requests when the region state has changed (transferring leader, splitting, merging, etc.). It's an optional optimization which helps avoiding useless waiting when the region state changes.

### New implementation of the scheduler command `AcquirePessimisticLock`

#### Handling write conflicts

As we already mentioned before, it will be allowed to acquire pessimistic locks even if there's a newer version with its `commit_ts` greater than the request's `for_update_ts`. When this behavior occurs:

- The lock will be written with the `for_update_ts` field equal to the `commit_ts` of the latest version
- The `max_ts` (for async commit) may be updated to the `for_update_ts` field of *the request* as usual, but do not update it to the latest `commit_ts` (which is also the `for_update_ts` field of the actually written lock).

#### Waiting and resuming

When a request encounters another lock, it starts waiting. Then there are two different ways how the request finishes:

1. The previous lock is released, and the request is resumed. It continue trying to lock the key and returns after locking successfully. The callback that returning response to client will be called after the resumed scheduler command is finished.
2. It waits for too long and exceeds the timeout. The callback will be called in `WaiterManager`.

So once a pessimistic lock request enters waiting state, it's callback will be shared. One of scheduler and waiter manager will take the callback and consume it.

### The client

Since a pessimistic lock request may lock a key when there's a version newer than the current snapshot being read, the client cannot simply use the new locking behavior as before. The client will retry the current operation (for TiDB, it's the current statement) while keeping the lock, and unlock it only if it's found that the key don't need to be locked anymore during the retry.

For example, assuming the client is TiDB and it's executing a statement at `for_update_ts = 10`, but a version whose `commit_ts` is 15 is found when locking one of the keys (we denote this case by `locked_with_conflict_ts = 15`), we cannot continue executing like if there isn't the conflict, because the statement may produce a totally different result if it's executed at `for_update_ts = 15`. Therefore, the client still need to retry the statement with a newer `for_update_ts`. However, the difference is that, the keys that were locked during the previous attempt will be kept locked. We expect that in most of the cases, the keys that need to be locked won't change (or won't change much) when retrying the statement. The benefit is that, it prevents the key from being locked by another transaction while the statement is retrying, causing the retry useless. In case that the keys that need to be locked has changed after retry, we acquire locks that we need but were not locked in the last attempt, and unlock the keys that were locked in the last attempt but no longer needed now.

### Updating lock-wait info for queueing requests

When many requests are queued waiting the same key, and one of them is granted the lock after the previous lock being released, then for the other queueing requests, the transaction they are waiting for is changed. In this case, we need some update operations so that it can perform correct deadlock detection and provide diagnostic information about lock waiting. The problem doesn't exist in the old implementation, because the queueing requests will all be canceled (and they then will retry) after delaying for `wake-up-delay-duration`.

We store the information about the owner of the current lock, along with the lock waiting queue of each key. When the lock's owner changes, the information about the current lock owner should be updated, and all entries in the queue should be iterated to collect information for performing deadlock detection.

### Compatibility

We need to support mixing pessimistic requests that choose to use or not to use the new behavior, since we are going to support the new behavior only for single-key requests. It's also important when considering rolling updating (old TiDB nodes may run with new TiKVs).

In the new implementation, requests using both new or old behavior will be queued in the new lock waiting queues. The difference will be the logic of waking up. When a scheduler command is releasing a lock and thus waking up a suspending pessimistic lock request, it follows the logic as shown in the pseudo code below:

```rust
if queue.front().lock_after_woken_up { // i.e. the head of the queue is a request using new behavior.
    queue.pop().wake_up_and_resume_execution();
} else {
    queue.pop().wake_up_and_report_write_conflict();
    // Delay for wake_up_delay_duration, and wake up as many legacy requests as possible,
    // until the queue is cleared or a request using new behavior is found.
    await_for_wake_up_delay_duration();
    while !queue.is_empty() && !queue.front().lock_after_woken_up() {
        queue.pop().wake_up_and_report_write_conflict();
    }
    if !queue.is_empty() {
        // Then the head of queue must be using the new behavior.
        // It's possible that the key is locked by another transaction after waking up the first head of queue.
        // If so, this request will find that lock and continue waiting.
        queue.pop().wake_up_and_resume_execution();
    }
}
```

## Drawbacks

The changes will be very complicated, especially considering the compatibility. For users that don't enable the feature, though we expect the behavior is totally unchanged, the internal implementation of lock waiting and waking up will be migrated to the new one, leading to risk to introduce new bugs and performance regression.

Though this feature is helpful in reducing tail latency in high contention scenarios, however it's at the expense of higher average latency. In our demo tests, in some extreme cases, the regression of average latency of enabling the feature can be up to 12%. We've analyzed the reason of the regression and has idea that's likely to be able to optimize it out, but it will be a hard work.

## Alternatives

&hyphen;

## Unresolved questions

### `PessimisticRollback`

`PessimisticRollback` is performed when a statement retires of fails, or the user rollbacks the whole transaction. Currently, a `PessimisticRollback` request carries a `for_update_ts`, and removes pessimistic locks with `for_update_ts` less than or equal to the specified `for_update_ts`. If a lock has larger `for_update_ts`, the lock is regarded to be newer than the `PessimisticRollback` request, thus it won't be removed.

Thing becomes a little complicated as we are supporting locking with conflict. When the locking-with-conflict behavior occurs, a lock with `for_update_ts` larger than that specified by `PessimisticLock` request might be written.

For example, TiDB executes a statement with `for_update_ts = 10` which locks several keys, one of which has `locked_with_conflict_ts = 15`. Let's denote the key that has conflict by `K`. Then consider the case that it needs to perform a `PessimisticRollback`. Of course, if we still pass `for_update_ts = 10` in `PessimisticRollback` request and handle the request in the same way as before, the lock on key `K` won't be able to be removed since it has a larger `for_update_ts`.

An intuitive way to solve the problem is to use the max `locked_with_conflict_ts` (which is 15 in this example) as the `for_update_ts` field in `PessimisticRollback` request. However, considering there is Async Commit and 1PC, there's still a corner case. If the previous version of key `K` is committed in async commit protocol, then its `commit_ts` (which is 15) might be calculated by some PD-allocated ts (14) plus one. Then the next attempt of the statement may also get 15 from PD as the new for_update_ts. As a result, if we pass `for_update_ts = 15` in `PessimisticRollback`, it might mistakenly remove the locks that's expected to be acquired during the new turn of retry.

We don't need to solve the problem immediately since if we choose to use the max `locked_with_conflict_ts` in `PessimisticRollback`, the possibility of the ts collision problem seems to be low. To solve it completely, here is two possible solutions:

#### Solution 1

Add a field to the lock that indicates the statement index / operation index, which is increased by one for every statement, including retry. The `PessimisticRollback` should also carry such an index. If a `PessimisticRollback` request finds a lock with greater index, it won't do anything to the lock. However, this approach requires changing the data format of the lock, which makes the solution risky. A benefit is that it may also be able to provide diagnostic information (by making it possible to know which statement in a transaction the lock belongs to).

#### Solution 2

Update the TSO interface to support allocating more than one timestamp at once. When performing pessimistic retry, we allocate two timestamps at once, and only use the larger one as the new `for_update_ts`. This avoids the new `for_update_ts` collides the ts that's being `PessimisticRollback`-ed.
