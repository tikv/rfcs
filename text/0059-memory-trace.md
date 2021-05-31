# Fully instrumented memory trace

## Summary

Make TiKV fully instrumented to provide accurate and details memory trace.

## Motivation

Suspicious memory increase has been a big challenge for TiKV. We don't have a
good tool to know who is exhausted the memory. With the help of profile tools,
we can identify memory leaks that can be reproduced and grow rapidly. However,
using profiling tools requires a recompilation and also has a performance
impact. And profile tool can only provide information about allocations, for
diagnostics, find out who is holding the memory is more useful.

Another use case is to limit the memory usage of a query. This is very useful
in implementing QoS and avoiding OOM.

## Detailed design

To calculate memory usage, we need to define a memory trace first.

```rust
pub struct MemoryTrace;

impl MemoryTrace {
    pub fn add_sub_trace(&mut self, name_id: u64) -> &mut MemoryTrace;
    pub fn set_size(&mut self, size: usize);
}
```

A memory trace is tree that records how much memory its children and itself
uses. It doesn't need to match any function stacktrace, instead it should have
logically meaningful layout. For example, memory usage should be divided into
several components under the root scope: TiDB EndPoint, Transaction, Raft,
gRPC etc. TiDB EndPoint can divide its children by queries, while Raft can
divide memory by store and apply. Name are defined as number for better
performance. In practice, it can be mapped to enumerates instead.

`size` should be the memory the category holds. It should *own* the memory.

To record a trace, we need to define providers.

```rust
trait MemoryTraceProvider {
    fn trace(&mut self, dump: &mut MemoryTrace);
}
```

The trace method will fill the trace with stats itself and its children have.

To trigger all providers, we need a trace manager.

```rust
pub struct MemoryTraceManager;

impl MemoryTraceManager {
    pub fn snapshot(&self) -> MemoryTrace;
    pub fn register_provider(&mut self, provider: Box<dyn MemoryTraceProvider + Send + 'static>);
}

```

`register_provider` will store the provider and `snapshot` will create a trace
using all the information from registered providers.

### Output

A trace is a rust struct, when dumping the result, it needs to be transformed
into some readable format. For best readability, it can be output as a
framegraph using the real name. It can also support the existing format from
[minitrace][1] to utilize more existing UI front end.

[1]: https://github.com/pingcap-incubator/minitrace-rust

### Implementation example

#### Tracing memory used by peer fsm

The stats can be stored in `StoreMeta`, and be updated after infrequent
existing tick or a specific new tick. We define an additional struct as
Provider:

```rust
pub struct RaftStoreMemoryTraceProvider {
    meta: Arc<Mutex<StoreMeta>>,
}

impl MemoryTraceProvider for RaftStoreMemoryTraceProvider {
    fn trace(&mut self, dump: &mut MemoryTrace) {
        let info = {
            let m = meta.lock().unwrap();
            m.memory_stats.clone()
        };
        for (peer_id, stat) in info {
            // record stats.
        }
    }
}
```

#### Tracing memory used by grpc

gRPC already records its memory usage in quota, all we need to do is querying
the quota.

```rust
pub struct GRpcMemoryTraceProvider {
    quota: MemoryQuota,
}

impl MemoryTraceProvider for GRpcMemoryTraceProvider {
    fn trace(&mut self, dump: &mut MemoryTrace) {
        dump.set_size(quota.used());
    }
}
```

## Drawbacks

Instrumentation can make code complicated.

## Alternatives

We can still keep using the profile tool we used to do and delivering profile
build by default. It can have uncontrolled performance impact and doesn't have
a clear view on historical allocations.

## Unresolved questions

How to ensure all memory allocations are traced?
