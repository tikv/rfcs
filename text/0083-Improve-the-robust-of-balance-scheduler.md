# Improve the robust of balance scheduler

- RFC PR: [https://github.com/tikv/rfcs/pull/85](https://github.com/tikv/rfcs/pull/83)
- Tracking Issue: [https://github.com/tikv/pd/issues/](https://github.com/tikv/pd/issues/4428)

## Summary

Make scheduler more robust for dynamic region size.

## Motivation

We have observed many different situations when the region size is different. The major drawback comes from this aspects:

1. Balance region scheduler pick source store in order of store's score, the second store will be picked after the first store has not met some filter or retry times exceed fixed value, this problem is also exist in target pick strategy.
2. Operator has an import effect on region leader, and the leader is responsible in the operator life cycle. But the region leader will not be limited by any filter.
3. There are some factor that influence execution time of operator such as region size, IO limit, cpu load. PD needs to be more flexible to manage operator's life not fixed config.
4. PD should know some global config about TiKV like `region-max-size`, `region-report-interval`. This config should synchronize with PD.

## Detailed design

### Store pick strategy

It can arrange all the store based on label, like TiKV and TiFlash and allow low score group has more chance to scheduler. But the first score region should has highest priority to be selected.

#### Consider Influence to leader

Normally, one operator is made of region, source store and target store, the key works finished by region leader such as snapshot generate, snapshot send. It is not friendly to the leader if majority operator is add follow.

It will add new store limit as new limit type to decrease leader loads of every store.

### Operator control

#### Store limit cost

Different size region occupy tokens should be different. Maybe can use this formula:

![](https://latex.codecogs.com/gif.image?\dpi{200}&space;\bg_white&space;Influence=\sum_{i=0}^{j}step_{i}.Influence&space;\newline&space;Cost&space;=&space;200*ln{\frac{region_{size}}{100KiB}})

Cost equals 200 if operator influence is 1Mb or equals 600 if operator influence is 1gb.

#### Operator life cycle

The operator life cycle can divide into some stages: create, executing(started), complete. PD will check operator stage by region heart beats and cancel operator if one operator‘s running time exceed the fixed value(10m).

It will be better if we can calculate every step expecting execute duration by major factor includes region size, IO limit and operator concurrency like this:

![](https://latex.codecogs.com/gif.image?\dpi{200}&space;\bg_white&space;V=\frac{io_limit}{sending_{count}+receiving_{count}}=\frac{100Mb/s}{3+3}=16.7Mb/s\newline&space;T_{transfer}=\frac{10Gb}{16.7Mb/s}=598s\newline&space;T_{total}=T_{generator}+T_{transfer}+T_{apply})

The snapshot generator duration can ignore because it doesn't need to scan. The apply snapshot duration will finish in minute level if it needs to load hot buckets.

### Sync global config

There are some global config that all components need to synchronize like `region-max-size`, `io-limit`. Using ETCD api to implement global config may be a good idea like [this](<[https://github.com/pingcap/tidb/pull/31010/files](https://github.com/pingcap/tidb/pull/31010/files)>)

## Drawbacks

## Alternatives

Removing peer may not influence the cluster performance, it can be replace by leader store limit.

Canceling operator can depends on TiKV not by PD, but TiKV should notify PD after canceled one operator.

## Questions

## Unresolved questions
