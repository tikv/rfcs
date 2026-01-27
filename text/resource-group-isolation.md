# Design Doc: Fair Scheduling Based on Historical RU Consumption

## Summary

This document proposes fair scheduling for TiKV resource control, prioritizing tenants based on historical RU consumption to protect sustained traffic from new spikes.

### Goal

Identify and throttle traffic causing overload based on historical RU consumption, while protecting sustained workloads.

### Non-Goals

This will be implemented for read traffic first. Write traffic will continue using the existing design. 

### Current State

TiKV implements resource control at the **resource group level**:
- Resource groups represent tenants and track RU (Resource Unit) consumption
- Users can define a quota while creating a resource group. Tenants with higher quotas are allocated proportionally more resources in TIKV 
- Each resource group also has a `group_priority` (LOW, MEDIUM, HIGH)
- The `ResourceController` uses mClock algorithm with virtual time (VT) per resource group
- VT is incremented proportional to the resources consumed
- Each resource group has a weight derived from proportional quota (`max_quota / tenant_quota`), where quota is the RU_PER_SEC defined when creating the resource group; VT increments are multiplied by this weight factor
- Tasks are ordered by: `concat_priority_vt(group_priority, resource_group_vt)`

## Problems

1. Customers typically don't know their RU usage in advance, so quotas are often over-provisioned to avoid throttling. As a result, scheduling based on static quota allocation fails to throttle the traffic actually causing system overload.    
2. The current approach increments VT by `consumed * weight`, where weight is derived from RU quota. This doesn't work because tenants with small QPS can still exceed their proportional quota without being throttled—their VT increment remains small, so they continue to be served even when causing overload. 

### Problem Scenario

**New traffic spikes should not impact sustained traffic.**

```
Steady state:
- Tenant_1: consuming 10000 RU/s (sustained workload)
- Tenant_2: consuming 20000 RU/s (sustained workload)
- System: stable

Sudden spike:
- Tenant_3: traffic suddenly increases to 5000, overloading the system

Expected:
- Throttle Tenant_3 (the new traffic causing overload)
- Protect Tenant_1 and Tenant_2 (sustained traffic)
```

## Design

### Two-Phase Priority Scheduling

Each tracker maintains two tags:

| Tag | Purpose | Update Formula |
|-----|---------|----------------|
| **Reservation Tag (R)** | Guarantees minimum throughput | `R += consumed / reservation` |
| **Weight Tag (W)** | Proportional sharing of excess | `W += consumed * weight` |

**Scheduling phases:**
- **Phase 0**: Entity has not received its guaranteed minimum → use R-tag (highest priority)
- **Phase 1**: Entity has received minimum → use W-tag for proportional sharing

**How this solves the problem:**
- Sustained traffic (Tenant_1, Tenant_2) stays below reservation → phase 0 (high priority)
- New spike (Tenant_3) quickly exceeds reservation → phase 1 (lower priority)
- When queue is full, phase 1 requests get evicted first

### Historical RU Tracking

Track RU consumption over a 10-minute sliding window using 1-minute buckets. This provides:
- Tenants with consistent historical consumption get higher reservation
- New or bursty traffic has low historical average, gets lower reservation
- Reservation adjusts based on actual usage, not static configuration

### Scenarios Fixed by This Implementation

**Scenario 1: New Tenant Spike**
```
Tenant_3 traffic increases → exceeds reservation → enters phase 1
Tenant_1 and Tenant_2 stay below reservation → remain in phase 0
```
Tenant_3 will be deprioritized if system becomes overloaded.

**Scenario 2: Hot Region Moved to TiKV Node**
```
Hot region moved to this node → Tenant_1 traffic increases → exceeds reservation → enters phase 1
Tenant_2 and Tenant_3 stay below reservation → remain in phase 0
```
Tenant_1 will be deprioritized if system becomes overloaded. If system is not overloaded, all tenants continue normally despite phase differences.

**Scenario 3: New Region Moved to TiKV Node (Gradual Ramp-Up)**
```
New region moved to this node → Tenant_1 traffic gradually increases
Historical RU consumption grows → reservation adjusts upward → Tenant_1 stays in phase 0
```
Sustained traffic is protected because reservation adapts to historical usage.

## Implementation

### Priority Encoding

Uses existing 64-bit priority format:

```
┌──────────────────────────────────────────────────────────────┐
│                      64-bit priority                          │
├──────────┬──────────┬────────────────────────────────────────┤
│  4 bits  │  4 bits  │              56 bits                   │
│  group   │  phase   │           tag value                    │
│ priority │  (0/1)   │                                        │
└──────────┴──────────┴────────────────────────────────────────┘

Phase 0: Reservation not satisfied → HIGHEST PRIORITY
Phase 1: Proportional sharing → NORMAL PRIORITY
```

### Historical RU Tracking

Track RU consumption over a 10-minute sliding window:

```
struct RuTracker:
    buckets: [u64; 10]          # 10 x 1-minute buckets
    current_bucket: usize
    last_rotation_time: u64

function record(ru):
    maybe_rotate_bucket()
    buckets[current_bucket] += ru

function get_rate_per_second():
    total = sum(buckets)
    return total / 600.0        # 10 minutes = 600 seconds

function maybe_rotate_bucket():
    now = current_time()
    if now - last_rotation_time >= 60 seconds:
        current_bucket = (current_bucket + 1) % 10
        buckets[current_bucket] = 0
        last_rotation_time = now
```

### Extending GroupPriorityTracker

Extend the existing `GroupPriorityTracker` struct with new fields:

```
struct GroupPriorityTracker:
    # Existing fields
    ru_quota, group_priority, weight
    virtual_time                    # Weight tag (phase 1)
    vt_delta_for_get

    # NEW fields (only used by read controller)
    reservation_tag                 # Phase 0 tag
    ru_tracker                      # 10-minute sliding window

# Reservation computed on the fly:
#   if enable_dynamic_reservation: computed dynamically
#   else: ru_quota
```

**Key design decisions:**
- In the current implementation, Read and write controllers have **separate instances** of `GroupPriorityTracker`
- New fields only updated by **read controller**
- Write controllers continue using just `virtual_time` as before

### Two-Phase Priority Calculation

```
function get_priority_two_phase(tracker):
    # Increment both tags (like existing vt_delta_for_get)
    tracker.virtual_time += vt_delta_for_get
    tracker.reservation_tag += r_tag_delta_for_get

    r_tag = tracker.reservation_tag
    w_tag = tracker.virtual_time

    if r_tag <= last_min_r_tag:
        # Phase 0: Has not received guaranteed minimum
        return encode_priority(group_priority, phase=0, r_tag)
    else:
        # Phase 1: Proportional sharing
        return encode_priority(group_priority, phase=1, w_tag)

function consume(tracker, ru):
    # Update weight tag (existing)
    tracker.virtual_time += ru * weight

    # Update reservation tag
    reservation = get_reservation(tracker.ru_quota)
    tracker.reservation_tag += ru / reservation

    # Record for historical tracking
    tracker.ru_tracker.record(ru)
```

### ResourceController Changes

```
function add_resource_group(name, ru_quota, priority):
    tracker = GroupPriorityTracker {
        ...existing fields...
        reservation_tag: last_min_vt,
        ru_tracker: RuTracker::new(),
    }

# Update TaskPriorityProvider implementation
impl TaskPriorityProvider for ResourceController:
    function priority_of(extras):
        tracker = get_tracker(extras.group_name)
        if is_read:
            return get_priority_two_phase(tracker)
        else:
            return get_priority(tracker)  # Existing behavior
```

### Periodic Maintenance

Extend existing `update_min_virtual_time()`:

```
function update_min_virtual_time():
    # Find min/max for both tags
    for tracker in all_trackers:
        update min_vt, max_vt
        if is_read:
            update min_r_tag, max_r_tag

    # Pull lagging trackers forward
    for tracker in all_trackers:
        if tracker.vt < max_vt:
            tracker.vt += (max_vt - tracker.vt) / 2
        if is_read and tracker.r_tag < max_r_tag:
            tracker.r_tag += (max_r_tag - tracker.r_tag) / 2

    # Reset if near overflow
    if max_vt > OVERFLOW_THRESHOLD:
        subtract OVERFLOW_THRESHOLD from all tags

    # Store min values for phase determination
    last_min_vt = max_vt
    if is_read:
        last_min_r_tag = max_r_tag  # Used in get_priority_two_phase
```

Load shedding is enforced through **queue eviction** - when the queue is full, lower priority tasks (phase 1) are evicted in favor of higher priority tasks (phase 0).

## Alternative Design: Dynamic Weight Adjustment (No Reservation Tag)

Instead of adding `reservation_tag`, modify existing weight calculation based on historical usage:

```
overload_ratio = ru_tracker.get_rate() / ru_quota
effective_weight = weight * max(1, overload_ratio)
vt_delta = consumed * effective_weight
```

**Pros:**
- Simpler - no new tag, reuses existing VT logic

**Cons:**
- May not protect sustained traffic from spikes effectively

## Configuration

```toml
[resource-control]
# Enable dynamic reservation (default: false)
# When false: reservation = ru_quota
# When true: reservation computed dynamically based on historical usage
enable-dynamic-reservation = false
```
