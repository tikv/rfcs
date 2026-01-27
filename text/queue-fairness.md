# Design Doc: Queue Fairness in TiKV Unified Read Pool

## Summary

This document proposes queue fairness mechanisms for TiKV's unified read pool, ensuring fair resource distribution across tenants and background traffic. The design introduces queue eviction and background task demotion.

## Motivation

### Current State

TiKV's unified read pool uses yatp's priority queue (SkipMap-based):
- Tasks are ordered by `concat_priority_vt(group_priority, group_vt)`
- Queue has a maximum size (`max_tasks_per_worker * worker_count`)
- When queue is full, new tasks are rejected with `ServerIsBusy`

### Goals

1. **Priority-based Eviction**: Higher-priority incoming tasks should evict lower-priority queued tasks
2. **Background Task Demotion**: Background tasks should never block foreground traffic

## Design

### 1. Background Task Demotion

Background tasks (GC, compaction, statistics) use LOW `group_priority` regardless of their resource group's configured priority:

```rust
// Force LOW priority for background tasks
let group_priority = if metadata.is_background() {
    LOW  // Always LOW, ignoring resource group config
} else {
    resource_group.priority  // Use configured priority
};
```

This ensures foreground traffic is always prioritized over background work.

### 2. Queue Eviction

When the queue is full, incoming high-priority tasks can evict low-priority queued tasks.

**Algorithm**:
1. Calculate priority of incoming task
2. Compare with lowest priority task in queue
3. If incoming has higher priority: evict lowest, enqueue incoming
4. Else: reject incoming with `ServerIsBusy`

Evicted tasks are failed with `ServerIsBusy` error.

### Priority Comparison

Priority is compared hierarchically:

```rust
struct TaskPriority {
    group_priority: u8,   // 1-16 (HIGH/MEDIUM/LOW)
    group_vt: u64,        // Resource group virtual time
    region_vt: u64,       // Region virtual time (optional)
}

impl Ord for TaskPriority {
    fn cmp(&self, other: &Self) -> Ordering {
        // Higher group_priority = higher priority
        // Lower group_vt = higher priority
        // Lower region_vt = higher priority
        self.group_priority.cmp(&other.group_priority).reverse()
            .then(self.group_vt.cmp(&other.group_vt))
            .then(self.region_vt.cmp(&other.region_vt))
    }
}
```

## Implementation

### 1. TaskMetadata Changes

Add `is_background` field:

```rust
const IS_BACKGROUND_MASK: u8 = 0b0000_1000;

impl TaskMetadata {
    fn is_background(&self) -> bool {
        self.mask & IS_BACKGROUND_MASK != 0
    }
}
```

### 2. yatp Modifications

**Add eviction support to priority queue**:

```rust
impl QueueCore {
    fn try_evict_for_priority(&self, incoming_priority: TaskPriority) -> bool {
        if let Some(lowest_entry) = self.pq.back() {
            if incoming_priority < lowest_entry.priority {
                // Evict lowest priority task
                if let Some(entry) = self.pq.pop_back() {
                    // Send eviction signal via oneshot channel
                    entry.eviction_handle.evict();
                    return true;
                }
            }
        }
        false
    }
}
```

**Update MapKey for priority queue**:

```rust
struct MapKey {
    priority: TaskPriority,
    sequence: u64,
}
```

### 3. Evictable Future Wrapper

Wrap futures with eviction notification:

```rust
struct EvictableFuture<F> {
    future: F,
    eviction_rx: oneshot::Receiver<()>,
}

impl<F: Future> Future for EvictableFuture<F> {
    type Output = Result<F::Output, ServerIsBusy>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // Check eviction signal first
        if self.eviction_rx.try_recv().is_ok() {
            return Poll::Ready(Err(ServerIsBusy));
        }

        // Poll inner future
        self.future.poll(cx).map(Ok)
    }
}
```

### 4. ReadPoolHandle Changes

Update `spawn()` to support eviction:

```rust
impl ReadPoolHandle {
    fn spawn(&self, ...) -> Result<(), ReadPoolError> {
        // 1. Calculate approximate priority (without VT increment)
        let approx_priority = resource_ctl.approximate_priority_of(&extras);

        // 2. Check if queue is full
        if running_tasks >= max_tasks {
            // 3. Try eviction
            if !remote.try_evict_for_priority(approx_priority) {
                return Err(ReadPoolError::UnifiedReadPoolFull);
            }
        }

        // 4. Create eviction channel
        let (eviction_tx, eviction_rx) = oneshot::channel();

        // 5. Wrap future with eviction support
        let evictable_future = EvictableFuture {
            future: task_future,
            eviction_rx,
        };

        // 6. Spawn task with eviction handle
        remote.spawn_with_eviction(task_cell, eviction_tx);

        Ok(())
    }
}
```

### 5. Priority Calculation with Background Demotion

```rust
impl TaskPriorityProvider for ResourceController {
    fn priority_of(&self, extras: &Extras) -> TaskPriority {
        let metadata = TaskMetadata::from(extras.metadata());

        // Get base priority from resource group
        let base_priority = self.resource_group(metadata.group_name())
            .group_priority;

        // Demote background tasks to LOW priority
        let group_priority = if metadata.is_background() {
            LOW
        } else {
            base_priority
        };

        let group_vt = self.resource_group(metadata.group_name())
            .get_group_vt();

        TaskPriority {
            group_priority,
            group_vt,
            region_vt: 0,  // Or from region tracker if enabled
        }
    }

    fn approximate_priority_of(&self, extras: &Extras) -> TaskPriority {
        // Read VT without incrementing (for eviction check)
        // Used to compare incoming task priority with queued tasks
    }
}
```

## Eviction Scenarios

### Scenario 1: High-Priority Foreground Evicts Low-Priority Background

```
Queue: [BG_task(LOW, vt=100), BG_task(LOW, vt=200), ...]
Incoming: FG_task(HIGH, vt=50)

Result: BG_task(LOW, vt=200) evicted, FG_task enqueued
```

### Scenario 2: Same Priority, Lower VT Evicts Higher VT

```
Queue: [task_A(MEDIUM, vt=500), task_B(MEDIUM, vt=600), ...]
Incoming: task_C(MEDIUM, vt=100)

Result: task_B(MEDIUM, vt=600) evicted, task_C enqueued
```

### Scenario 3: Incoming Cannot Evict

```
Queue: [task_A(HIGH, vt=100), task_B(HIGH, vt=200), ...]
Incoming: task_C(LOW, vt=50)

Result: task_C rejected with ServerIsBusy
```