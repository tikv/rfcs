# RFC: Load-based Region Splitting

## Summary

This proposal presents a load-based region splitting method that automatically
splits region when it remains a read hotspot continuously. After splitting, the
hotspot can be easily scattered through Region scheduling.

## Motivation

The existing scheduling mechanism (`hot-region-scheduler` in TiKV 3.0) supports
distributing multiple hot read regions on the same node to reduce the load on
the node. However, if all read hotspots are in the same region, this mechanism
cannot further distribute hotspots. A temporary approach is to manually split
the region and suspend the region merge, but its operation cost is high and it
has negative impacts on non-hotspots.

CockroachDB introduced Load-Based Splitting to solve this problem. After
investigation, we think this method is also applicable to TiKV clusters, so
reference will be made to its functional design.

## Design

### Design Considerations

* There is a cost to perform a split. And after splitting, hotspots can only be
  solved after the leaders of newly generated multiple regions have been
  distributed to different nodes. In this case, splitting is meaningless for
  momentary and short  loads（< 10s). Therefore, load-based splitting should be
  performed only after intense traffic generating in the region lasts for a
  period of time.
* After Region has been split into two new regions, the split regions should be
  prevented from being merged immediately. Therefore, the overall loads of the
  two regions should be checked before the region merge.
* To avoid back-and-forth splitting and merging, the merging load threshold
  should be slightly lower than splitting load threshold (e.g., 20% lower).

### Detailed Design

1. When the read load of a region exceeds the splitting threshold, start
   counting the traffic of keys. Through probability sampling of requests on
   this region, 20 keys are randomly selected first.
2. For each key selected above, count the following values for 10 seconds. If
   the load falls below the threshold within this period of time, the process
   will be terminated.
    * The number of times only the data to the left of the key was accessed
      (excluding the key)
    * The number of times only the data to the right of the key was accessed
      (including the key)
3. According to the above statistics, exclude the following special cases first.
    * The splitting is meaningless if all hotspots are on one key. In this case,
      you can find over 99% of the accesses to all selected keys are on one
      side. At this time, mark the region to keep it from being automatically
      split for a period of time and then terminate the process.
4. According to the above statistics, select the optimal split point from the 20
   keys. The number of accesses to the left and right sides of the optimal
   splitting point should be as balanced as possible.
5. Split the region at the optimal splitting point. And inform the region
   scheduler to distribute the load as soon as possible.

## Drawbacks

If TiKV is reading keys sequentially, region split may be triggered incorrectly.

## Alternatives

`Follower Read` feature can also distribute hotspots. Increasing the number of
replicas of follower as needed can support a larger read capacity. However,
increasing the number of followers will reduce the writing efficiency. At the
same time, follower reads need to get read index from the region leader, which
has extra costs compared with leader reads. Therefore, to distribute hotspots
with `Follower Read` is more suitable for scenarios where read requests are
performed in a large scale.

## Unresolved questions

How to optimize the algorithm of the splitting point has not been determined
yet. We will do a POC to see whether it can pass the tests of some typical
scenarios first, and then compare different methods before we make the final
decision.
