# Load-Based Replica Read

- RFC PR: https://github.com/tikv/rfcs/pull/105
- Tracking issue: (TODO)

## Summary

When the read pool load of the leader TiKV is too high, it can reject a coming read request. After that, the client will send the request to follower or learner peers to retry the request in replica-read mode. When a read hotspot occurs, this method allows us to make better use of cluster computing resources.

## Motivation

Some users may experience temporary store-level read hotspots. The read pool queues of some TiKVs become long and the read latency increases dramatically.

For example, an index lookup join in TiDB can issue a large number of discrete point queries. These queries may not be uniformly distributed, causing an imbalance across TiKV nodes. Such kinds of hotspots don't last for a long time and don't necessarily appear on a fixed TiKV node. In such a case, it would be impossible to resolve the hotspot using PD scheduling.

Therefore, we need a mechanism to increase resources quickly for the read hotspots.

## Detailed design

The resources for reading (read pool CPU and IOPS) on each TiKV node are limited. When a read hotspot appears, the resources of one TiKV can be easily exhausted but the whole cluster tends to be mostly idle.

Replica read, which enables us to read from follower and learner peers, is a feature that can extend resources instantly. However, to preserve linearizability, we have to know the latest index of the leader before reading at a non-leader peer, making replica read not a good choice in all cases.

Therefore, we hope to enable replica read intelligently based on the load of the leader. When all the read pools are vacant, all requests will still be handled by the leader for low latency. But if the read pool of some TiKV is too busy to handle requests in time, the client will automatically switch to other peers to utilize extra resources.

### Estimating load of TiKV read pool

Traditionally, the load is ratio of the average queue length and the processing speed. But we care more about the latency and latency is a metric that is more comparable across nodes. So, we will use the wait duration to represent the load.

Knowing the current queue length $L$ and the estimated average time slice $\hat S$ of the read pool, we can estimate that the wait duration is $T_{waiting} =L \cdot \hat S$.

The current queue length is easily known. We can use EWMA to predict the average time slice in the short future. We update EWMA using the following formula every 200 milliseconds (which is the unit of $t$ in the formula):

$$
\hat S_{t+1}=\alpha \cdot Y_{t}+(1-\alpha) \cdot \hat S_{t}
$$

where $\hat S_{t+1}$ is the predicted average time slice length at $t+1$, $Y_{t}$ is the observed value at $t$ over the past 200 milliseconds, $\hat S_{t}$ is the previous predicted value, $\alpha$ is the weight and we will examine its appropriate value later.

When the total time used in the read pool is very small, we will skip updating the EWMA until the accumulated execution time reaches a threshold (e.g. 100ms) to avoid EWMA being affected by a small number of unrepresentative samples.

### Rejecting request on busy

The client decides the maximum waiting duration. If this field is set, TiKV can return `ServerIsBusy` early if the estimated waiting duration exceeds the threshold. This threshold is also configurable by the user.

```protobuf
message Context {
    ...

    uint32 busy_threshold_ms = 26;
}
```

If TiKV returns `ServerIsBusy` based on the current load, the estimated waiting duration and the current index on the leader will be also returned to the client.

```protobuf
message ServerIsBusy {
    ...

    uint32 estimated_wait_ms = 3;
    uint64 applied_index = 4;
}
```

Because we will retry in replica-read mode, we don't need the follower or learner to issue a read index RPC again after knowing the applied index.

The estimated waiting duration may be useful for the client to select a better TiKV for the retried request.

### Retry strategy

_It is hard to find a perfect strategy. It is expected that the strategy changes after further experiemnts. So, this part only describes a possible solution._

Without any other information, the client will select a random follower or learner to retry the request after the client receives a `ServerIsBusy`. The `busy_threshold_ms` in the retried request will be set to _2 times_ the one returned in `ServerIsBusy`. So, if the retried TiKV is _much_ busier than the leader, it will reject the request again. The factor is to balance the extra RPC number and the load difference between TiKVs.

When the retried request is rejected again, the client will select a not selected replica to retry again. The `busy_threshold_ms` should still be calculated from the leader response. If all followers and learners reject the request because they are much busier than the leader, just unset `busy_threshold_ms` and let the leader execute the request without checking its load.

#### Optimizing strategy with load info

If the non-leader replicas are all busier than the leader, with the default strategy, we need `replica_num + 1` RPCs for each request. This is definitely unsatisfactory. Therefore, we should try to avoid useless attempts to busy nodes.

We maintain the estimated waiting duration in the client. Every time the client receives a `ServerIsBusy` error with `estimated_wait_ms`, it updates the information for the store and records the current time.

```go
type Store sturct {
    ...
    
    estimatedWait     time.Duration
    waitTimeUpdatedAt time.Time
}
```

To make use of as many resources as possible, the load we predict should not be larger than the current load. Otherwise, we may skip a node that is already free for executing requests and not get the best performance.

We use `estimatedWait - (time.Now().Since(waitTimeUpdatedAt))` as the estimated waiting duration in the client. It's mostly certain that this estimated value is smaller than real because the TiKV accepts requests meanwhile and some queries don't finish in a single time slice.

Now, `busy_threshold_ms` in the leader request may be increased to the minimum of the estimated waiting duration of all candidate TiKVs. This can reduce useless retries if all TiKVs are busy.

If the estimated waiting duration is greater than the threshold we are going to set, we can skip it and select the next possible node. In this way, if we know that a node is busy recently, we won't waste effort to send RPC to it.

And when selecting a node for replica read, we will sort them according to the estimated waiting duration maintained in the client. The node with shorter waiting duration will be sent first.

## Drawback

This RFC only considers the case when the bottleneck is at the read pool.

When load-based replica read is used, there will be many more useless RPCs. So, we will be giving more pressure to the gRPC part. We should be cautious not to enable it when gRPC is the bottleneck to avoid entering a victious circle.
