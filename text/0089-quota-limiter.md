# Quota limiter

- RFC PR: [https://github.com/tikv/rfcs/pull/89](https://github.com/tikv/rfcs/pull/89)
- Tracking Issue: [https://github.com/tikv/tikv/issues/12131](https://github.com/tikv/tikv/issues/12131)

## Summary

Add a global forefront throttle to limit read/write requests, which brings stable QPS.

## Motivation

On the machines where physical resources are constrained, the performance of TiKV may become unstable. Front-end and back-end processing interfere with each other. For example, TiKV's processing of background sampling will take up a lot of CPU processing time, which will cause other requests to be processed with less cpu resource. In addition, under long-term high load pressure, TiKV will accumulate more and more background tasks, such as compaction jobs, etc. When resources are limited and users are not sensitive to performance, it's better for TiKV to work stably under high pressure.

## Detailed design

### Limited method

This feature plans to add multiple global limiters into TiKV to record different types of processing speed. These types include forefront CPU time, forefront request rate, read/write bandwidth, and write KV rate.

When the limiter reaches the quota value, the request will be forced to block for a period of time to compensate, based on the result returned from a speed limiter using the token bucket algorithm.

Several new configs will be added. Users need to limit each metric separately, of which the quota for cpu time is an approximate value.

* quota.forefront-cpu-time: usize
* quota.forefront-req-rate: usize
* quota.write-kvs: usize
* quota.write-bandwidth: ReadableSize
* quota.read-bandwidth: ReadableSize

### Limited position

Clock and limit at the location of code blocks for the above metric that will significantly affect.

**scheduler write:** All txn write requests will be handled in the `process_write` function.

**tidb query executor:** Coprocessor DAG processing will be handled in `BatchExecutorsRunner`.

**txn get:** Get value processing will be handled in `storage::get`

**txn batch get:** Get multiple value processing will be handled in `storage::batch_get`

**analyze request:** Coprocessor analyze processing will be handled in `collect_columns_stats`.

## Drawbacks

Setting a quota that is too small may cause significant performance degradation, which requires experienced users to config the quota.
