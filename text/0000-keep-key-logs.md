# RFC: Keep Key Logs

- RFC PR: 
- Tracking issue:

## Motivation

To address https://github.com/tikv/tikv/issues/10841, we decide to prioritize availability over log completeness thus switch the overflow strategy of logger from `Block` to `Drop`. 

However, some logs are useful for diagnosis in practice. We heavily depend on logs to investigate bugs or abnormalities of the system. If all logs are dropped randomly, reasoning about the raftstore or transaction issues may be impossible.

Therefore, it is only a temporary workaround to simply drop overflowed logs. We hope to keep logs that are important for diagnosis if possible.

## Detailed design

The problem is that we don't define which logs are important. Whether a log is important for diagnosis is not related to the log level. 

For example, we prefer to keep info level logs in the raftstore rather than keep warn level logs in the coprocessor module. The former one is critical for finding out how each region evolves while the latter one may be only a retryable error.

So, these important logs deserve to have another set of macros:

- `key_crit!`
- `key_error!`
- `key_warn!`
- `key_info!`
- `key_debug!`
- `key_trace!`

Logs from these macros should not be dropped as they are key information. These macros should **NOT** be abused.

### Reserved space for key logs

We use async logging to reduce the performance impact on the working threads. The order of logs must be retained, so it is a good idea to still have only one channel to guarantee this property. That is to say, the key logs are pushed to the same channel as normal logs.

But we do not want normal logs to occupy all spaces of the channel. The solution is that we set a threshold for normal logs.

If the message number in the channel goes beyond half of the channel capacity, normal logs are refused and dropped. Key logs can be pushed to the channel until the channel is totally full.

In this way, we make sure at least half of the total capacity is always available to key logs.

### Overflow fallback

Despite the design of key logs, it is still possible we abuse key logs and make the channel full. 

The principle is that availability should be never sacrificed. So we must not block the sender on overflow, nor use an unbounded channel which has the risk of OOM. But we still hope to keep the key logs.

The fallback solution is to write logs to a file synchronously.

If key logs fail to be sent to the channel, they are formatted at the sender thread and written to the fallback log file. Without disk syncing, these writes should finish with very small latency and reasonable throughput. Then the working threads will not be badly affected and we do not really drop any key logs.

### Changes to `slog` modules

Key logs are distinguished by its tag. `key_info!(...)` is a macro equivalent to `info!(#"key_log", ...)`.

Add a new method `reserve_for_key_logs(self, fallback_path: &Path)` to `AsyncBuilder`.

If this option is set, the length of the channel should be checked in `<Async as Drain>::log` if the tag name is different from `"key_log"`. If the length is bigger than half the capacity, the log should be dropped.

## Unresolved questions

- Naming of the key log macros
- Channel capacity
- Configuration and filename of the fallback log file