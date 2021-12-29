# Improve the robust of balance region scheduler

- RFC PR: [https://github.com/tikv/rfcs/pull/85](https://github.com/tikv/rfcs/pull/83)
- Tracking Issue: [https://github.com/tikv/pd/issues/](https://github.com/tikv/pd/issues/4428)

## Summary

Make schedulers more robust for dynamic region size.

## Motivation

We have observed many different situations when the region size is different. The major drawback comes from these aspects:

1. Balance region scheduler picks source store in order of store's score, the lower store will be picked after the higher store has not met some filter or retry times exceed fixed value. If the count of placement rules or tikv is bigger, the lower store has less chance to balance like TiFlash.
2. splitting rocksDB and sending them by region leader will occupy cpu and io resources.
3. There are some factors that influence execution time of an operator such as region size, IO limit, cpu load. PD needs to be more flexible to manage operator's timeout threshold rather than not fixed value.
4. PD should know some global config about TiKV like `region-max-size`, `region-report-interval`. This config should synchronize with PD.

## Detailed design

### Store pick strategy

It can arrange all the stores based on label, like TiKV and TiFlash and allow low score groups more chances to schedule. But the first score region should have the highest priority to be selected.

#### Consider Influence to leader

It will add a new store limit to decrease leader loads of every store. Picking region should check if the leader token is available.

### Operator control

#### Store limit cost

Different size regions occupy tokens should be different. Maybe can use this formula:

![](https://latex.codecogs.com/gif.image?\dpi{200}&space;\bg_white&space;Influence=\sum_{i=0}^{j}step_{i}.Influence&space;\newline&space;Cost&space;=&space;200*ln{\frac{region_{size}}{100KiB}})

Cost equals 200 if operator influence is 1MB or equals 600 if operator influence is 1GB.

#### Operator life cycle

The operator life cycle can be divided into some stages: create, executing(started), complete. PD will check operator stage by region heartbeat and cancel if one operator‘s running time exceeds the fixed value(10m).

It will be better if we can calculate every step expecting execute duration by major factor includes region size, IO limit and operator concurrency like this:

![](https://latex.codecogs.com/gif.image?\dpi{200}&space;\bg_white&space;V=\frac{io_limit}{sending_{count}+receiving_{count}}=\frac{100Mb/s}{3+3}=16.7Mb/s\newline&space;T_{transfer}=\frac{10Gb}{16.7Mb/s}=598s\newline&space;T_{total}=T_{generator}+T_{transfer}+T_{apply})

The snapshot generator duration can ignore because it doesn't need to scan. The apply snapshot duration will finish in minute level if it needs to load hot buckets.

### Sync global config

There are some global config that all components need to synchronize like `region-max-size`, `io-limit`. Using ETCD api to implement global config may be a good idea like [this](<[https://github.com/pingcap/tidb/pull/31010/files](https://github.com/pingcap/tidb/pull/31010/files)>)

## Drawbacks

## Alternatives

Removing peer may not influence the cluster performance, it can be replaced by leader store limit.

Canceling operators can depend on TiKV not by PD, but TiKV should notify PD after canceling one operator.

## Questions

## Unresolved questions
