# Periodic Incremental Full Compaction

## Introduction

**Full compaction** is a scheduled task that runs on each TiKV node and compacts the column families of all regions at all levels including the bottommost (L6). In order to reduce to impact running such a heavy weight task would have on the traffic the cluster is serving as the task executes, two strategies are used:

1. Incremental: full  compaction runs incrementally on a range of keys (presently a region) at a time: 

2. To enable full compaction, start times must be specified: these are the times of the day during which the cluster is expected to be least loaded. 

## Motivation

Presently, we have no way to periodically execute compaction
across all levels and for all ranges. This has a number of
implications: compaction filters (used to remove tombstones)
only run when the bottom-most (L6) compaction is executed; a
system with high number of deletes that experiences a heavy
read-only (but little or no writes) might thus have accumulated tombstones markers that are not deleted.

## Detailed design

Full compaction runs as a background task using same mechanism as existing `Compact` and `CheckAndCompact`
tasks. 

```rust
PeriodicFullCompact {
        // Ranges, or empty if we wish to compact the entire store
        ranges: Vec<(Key, Key)>,
        compact_load_controller: FullCompactController,
    },
```

### Full compaction of a range

To fully compact a range of keys, we invoke 
### Periodic scheduling

### Incremental full compaction by region

#### Alternatives

#### Conditions for pausing

#### Pausing mechanism

### Metrics

## Future work

### Selecting the ranges

### Manual invocation

### Stopping compaction

* Add a mechanism to stop full compaction.