# cpu-threshold on unified read-pool

## Summary
The purpose of this design is to have upper bound on CPU utilization of read pool threads

## Motivation
In a cloud environment, some customers use EBS for data disks because of simplification of architecture. EBS latency is typically in the single-digit millisecond range. TiKV uses RocksDB in synchronous mode. To achieve better throughput with EBS-only instances, users often create more read pool threads than the total number of TiKV CPUs. However, creating more threads than the available cores can result in all CPU resources being allocated to read pool threads, which may starve other operations such as split/scatter and resolve-ts propagation. This increases the blast radius and recovery time during hot read scenarios, potentially overwhelming TiKV's CPU.

## Current implementation
TiKV monitors unified read pool statistics every 10 seconds(READ_POOL_THREAD_CHECK_DURATION). Based on these statistics, it adjusts the number of yatp threads by scaling up/down by one per epoch. The yatp framework handles scaling internally. It can create upto the maximum number of threads specified by the readpool.max-threads configuration but will not activate more threads than are currently needed. The size of the running task queue adjusts automatically based on the active thread count.

TiKV scale up the thread count when all the following conditions are met:
- The current number of threads is less than the maximum allowed.
- Adding an additional thread does not overload the CPU.
- All read pool threads are busy (threads’ busy time is at least 80%). It includes both blocking and running time.
- There are more than 3 tasks waiting in the running queue.

TiKV scale down thread count when all the following conditions are met:
- The current thread count exceeds the configured limit.
- The average thread utilization drops below the low water mark (70%).
- There are fewer than 3 running tasks in the scheduling queue.

## Detailed design
TiKV will monitor cpu stats of the read pool threads to scale up/down the number of threads.
TiKV scale down the thread count when all the following conditions are met:
- CPU usage of unified read pool is more than the readpool.unified.cpu-threshold
- It can bring the total number of threads less than readpool.unified.min-thread-count. 

TiKV scale up the thread count when all the following conditions are met
- CPU usage of unified read pool is less than the readpool.unified.cpu-threshold
- Total number of threads less than readpool.unified.min-thread-count

### Code changes

Code would be added to calculate the CPU usage of unified read pool threads

```
get_unified_read_pool_cpu() {
    // get all the read pool tids
    // thread::full_thread_stat() to get the cpu usage
}
```

adjust pool size

```
READ_POOL_THREAD_CHECK_DURATION 10 sec
adjust_pool_size() {
    if !self.auto_adjust {
        return
    }

    leeway = .1; // 10%
    if unifiedReadPoolCPU > (leeway + 1) * CPUTHreshold  {
        target_threads = max(cur_threads * (CPUThreshold * num_cores) / unifiedReadPoolCPU, CPUThreshold * num_cores);
        // scale down incrementally by target_threads;
        return
    }

    if unifiedReadPoolCPU < (leeway - 1) * CPUTHreshold && activeThreads < minThreadCnt {
        // scale up incrementally
        return
    }

    if (unifiedReadPoolCPU > (leeway - 1) * CPUThreshold && activeThreads >= minThreadCnt {
        // existing code to scale up/down decided by busy thread and running tasks
    }
}
```

### Scenarios
On Cloud, EBS env It is 10 cores machine
1. min_thread is 20
2. max_thread is 10 * 5 = 50
3. CPUTHreshold is 80%
4. current active thread count is 20

We will scale down the thread until cpu utilization of unified read pool reaches less than 8.8 cores. We will start doing scale up if CPU utilization is less than 7.2 cores and number of threads is less then 20.

## Configuration
readpool.unified.cpu-threshold : It represents the maximum cpu used by unified read pool. Its default config is set to 0 which indicates that there is no threshold.

## Benefit
This design could able to mitigate the high cpu usage due to hot reads in READ_POOL_THREAD_CHECK_DURATION * log(number_of_active_threads) 

## Alternatives

Alternative option is to run rocksdb in async mode. However, it is not a stable feature and even the existing implementation does not run completely in async mode.

