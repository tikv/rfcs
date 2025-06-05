# Enhance Slow Store Scheduler

- RFC PR: https://github.com/tikv/rfcs/pull/119
- Track issue: https://github.com/tikv/pd/issues/9359

## Summary

Added network status check in [Slow Store Scheduler](https://github.com/tikv/pd/blob/master/pkg/schedule/schedulers/evict_slow_store.go).

## Motivation

Assume a store in the cluster is experiencing network jitter. This jitter causes region leaders in the store to be re-campaigned by raft on other stores, resulting in leader transfers. However, PD is unable to detect the network jitter on the TiKV store, because PD can recieve the store heartbeat in this network status. Consequently, the PD's balance leader scheduler schedules the leader back to the problematic TiKV, leading to increased latency in the cluster.

To address the above problem, we need to add a new network status feedback mechanism to PD, and avoid to transfer leaders to the problematic TiKV node.

## Detailed design

### Slow Store Scheduler

Slow store scheduler is an existed scheduler in PD. Its current function is to detect whether the TiKV disk fails and quickly schedule the leaders away. Here is its steps:

1. Each TiKV collects the timeout I/O requests locally (during each inspect-interval)
2. Count `TimeoutRatio` and `SlowScore`
3. Send `SlowScore` to PD by store heartbeat
4. [pd side] If `SlowScore` >= 100, evict the leaders from this TiKV store.
5. [pd side] If `SlowScore` <= 1, recover this store
6. If `SlowScore` >= 100, a store heartbeat will be trigger at once to tell PD the store is slow.

In this RFC, we will enhance its detection of TikV network and continue to use its algorithm for calculating slow scores. However, we will not actively evict the leader from the tikv node with poor network. Instead, we will rely on the raft mechanism to let them elect a new leader on their own and ensure that the leader will not be actively dispatched back to the problem node by PD.

### NetworkSlowScore

`NetworkSlowScore` is a score that measures the network status of one store. The larger it is, the more likely there is a problem with the network. The value range of `NetworkSlowScore` is [1,100].

`NetworkSlowScore` is a score that rises quickly during a failure and falls smoothly after recovery. This allows PD to react quickly to continuous failures and allow tikv to resume its functionality after recovery.

#### How to count

When there is a problem with the network, the request sent by the store to other nodes(contain PD and TiKV) will time out, so we choose the request timeout rate as an indicator to measure the network status.

```go
init NetworkSlowScore = 1
for each InspectInterval:
    if TimeoutRatio.isNormal():
        NetworkSlowScore = NetworkSlowScore - (100 * (InspectInterval / MinTTR))
        // MinTTR: Minimal time that the score could be decreased from 100 to 1.
    else:
        // Normalize time out ratio
        normalizedTimeoutRatio = min(TimeoutRatio, RatioMaxThresh) / RatioMaxThresh
        // RatioMaxThresh: The maximal tolerated timeout ratio.
        NetworkSlowScore = min(100, NetworkSlowScore * (1 + growthFactor * normalizedTimeoutRatio))
```

#### Score increase

When some requests times out, the `NetworkSlowScore` starts to rise. When the `NetworkSlowScore` reaches 100, PD will think that there is a network problem with the tikv.
1. **RatioMaxThresh** is a maximum tolerance threshold for the timeout ratio. When it exceeds this value, we consider the store unavailable.
    > In the detection of disk io slow store, RatioMaxThresh was set to 0.1.
2. Next, the timeout rate is normalized.
    $$normalizedTimeoutRatio = min(TimeoutRatio, RatioMaxThresh) / RatioMaxThresh$$
3. Finally, `NetworkSlowScore` is enlarged according to the following formula
    $$NetworkSlowScore = min(100, NetworkSlowScore * (1 + growthFactor * normalizedTimeoutRatio))$$
    **growthFactor** is used to control the growth rate. 
    > In the detection of disk io slow store, it was set to 1. Need to be determined in subsequent discussions


#### Score decay

100 to 1 is a linear decrease over time. If the store's network is restored, we assume it will complete this process within **MinTTR**. Therefore, the formula for calculating the NetworkSlowScore is 
$$NetworkSlowScore = NetworkSlowScore - (100 * (InspectInterval / MinTTR))$$

> In the detection of disk io slow store, MinTTR is set to 5 min. Need to be determined in subsequent discussions

### How to collect TimeoutRatio

There are two ways to collect TimeoutRatio:

- Create a PD <-> TiKV health check mechanism. Directly use the information obtained by PD to determine the network status of TiKV. 
- Establish TiKV <-> TiKV health check mechanism. Each TiKV still uploads the health information obtained by the health check through store heartbeat.

#### PD <-> TiKV health check

PD <-> TiKV health check is equivalent to a higher-frequency store heartbeat. However, it is only responsible for health checks and does not mix other functions.

PD <-> TiKV health check is a very simple framework. However, it can't handle this situation that the problematic tikv is only experiencing network problems with other tikv nodes, but the network is normal with the pd nodes.

#### TiKV <-> TiKV health check

When a tikv has a network problem, other tikvs will also experience request timeouts when accessing the problematic tikv. In this case, the slow score of the normal tikv may also increase, and if it lasts for a long time, it may even reach 100, triggering the slow store mechanism.
To avoid this problem, we can send the collected health information and its corresponding store_id to PD. If the `TimeoutRatio` of a node exceeds RatioMaxThresh, it is considered that the network of the node has a problem, and PD will filter out the request timeouts from normal nodes and then calculate the `TimeoutRatio`.

Before, when `SlowScore` reaches 100, a heartbeat is sent directly to ensure the response time of the PD. In the network situation, we caculate the `NetworkSlowScore` int the PD. So if the failure rate of a round of health check exceeds `RatioMaxThresh`, a PD heartbeat is also sent directly. In this way, the network observation status of this PD can be seen in advance (although its score has not reached 100), and when the heartbeats of other stores are received later, the health check information of the store can be filtered out in time to avoid unnecessary noise.

#### Summary

Since the first method can quickly and effectively solve most scenarios, we will deliver the first method now and then complete the second method.

### Related Configuration

> The final default value may change with the test results.

`inspect-network-interval`:
- Type: Integer
- Default value: 100
- Unit: ms
- Range: [10, +∞]
- This parameter is used to set the inspect interval in the health check. Zero represents network inspect is disabled. It also represents the sensitivity and growth rate. When the `inspect-network-interval` is smaller, the more data is detected per unit time, and the score is likely to grow faster.

`network-recovery-duration`
- Type: Integer
- Default value: 5
- Unit: min
- Range: [1, +∞]
- This parameter is used to set the network recovery duration. 

`max-network-slow-stores`
- Type: integer
- Default value: 1
- Range: [0, +∞]
- This parameter indicates how many slow stores can exist in the cluster at the same time. When the NetworkSlowScore of more than n tikvs reaches 100, tikvs in excess of `max-network-slow-stores` will not be considered as slow stores.

`growth-factor` is not exposed as it is difficult to understand and quantify.

`inspect-network-interval` is a parameter in tikv-server, which is similar with [`inspect-interval`](https://docs.pingcap.com/tidb/dev/tikv-configuration-file/#inspect-interval). And `network-recovery-duration` can be set via `pd-ctl scheduler config evict-slow-store-scheduler set`.

The timeout for the health check is 1s.

## Drawbacks

PD <-> TiKV health check can't handle the situation that the problematic tikv is only experiencing network problems with other tikv nodes, but the network is normal with the pd nodes. TiKV <-> TiKV health check will resolve this problem.

All tikv scores go up when PD has network problems.

## Alternatives

Since network jitter can cause raft re-election and raft message accumulation, we may be able to determine the network status by monitoring these indicators. However, since TiKV uses the Multi-raft protocol, it is not easy to collect some raft-related data in the actual code implementation, and these data may also contain some noise, such as raft message persistence. We still choose health check as the basis for network judgment.