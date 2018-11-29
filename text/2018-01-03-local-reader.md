# Summary

At the time read requests are handled by a single thread named raftstore, which
also handles write requests and other tasks. This RFC proposes introducing a new
thread named local reader, for separating read requests from the raftstore.

# Motivation

There are three major workloads for the raftstore:

 - Handle read requests: CPU intensive.
 - Handle write requests: I/O intensive.
 - Drive Raft state machines: CPU intensive.

For read requests, TiKV takes a snapshot of the underlying RocksDB when a Raft
leader is in its lease. Read requests are lightweight and the raftstore can
handle them fast. However, due to the single-threaded character of the
raftstore, read requests may be blocked by other workloads. E.g.,

 - Read QPS drops while write requests get more, and the raftstore spends more
   time in writing Raft logs.
 - Read QPS drops while the Regions number grows, and the raftstore spends more
   time in driving Raft state machines.

By having a dedicated thread for read requests, we can separate read requests
from other workloads and address above issues. TiKV can get better performance
and lower latency on read most workload.

# Detailed design

The local reader offloads read requests from the raftstore and guarantee
linearizability.

## Local reader

The local reader uses ReadDelegates (delegate) to handle requests. Every
delegate is owned by a Raft peer which belongs to the raftstore. The delegate
and the peer communicate via a channel, and each pair of them shares an atomic
`LeaderLease`.

A peer can do local read as long as it holds the following conditions (only list
the most important):

 - It’s a Raft leader;
 - Its applied index’s term matches its current term;
 - It has a valid leader lease.

Before reading, a delegate checks the above conditions too. After reading is
done, it performs an extra lease check, because read operations have to be done
in a valid leader lease to guarantee linearizability. If a delegate fails these
checks, it redirects read requests to the raftstore.

## LeaderLease

`LeaderLease` is a state shared by the local reader and the raftstore. It is
critical to implement local reader correctly. When a leader peer steps down,
its delegate is no longer allowed to read, otherwise it may read the stall data
which violates linearizability. The shared `LeaderLease` is implemented by an
atomic variable, so a delegate observes leadership change immediately and stops
handling read requests.

## What requests can it handle?

The section specifies the requests that the local reader can handle. There are
three types of message defined in the [raft_cmdpb.proto].

 - Request
 - AdminRequest
 - StatusRequest

The local reader can only handle a subset of `Request`, that is

 - GetRequest
 - SnapRequest

As for other requests, it either redirects or panics on other requests.

## Corner cases

There are some corner cases and the typical ones are listed below. Most of the
them can be resolved by expiring atomic `LeaderLease`.

### Case 1

Local reader is blocked before taking the snapshot while the leadership has
changed.

It is addressed by expiring the leader lease. After reading, it will be checked
whether the lease is outdated. If yes, the reading results will not be returned
back.

### Case 2

Local reader is blocked before taking the snapshot while the target Region
splits into two. The original leader remains unchanged while the leader of the
new Region is elected on other TiKV.

It is addressed by expiring the original leader lease before split is done.

# Drawbacks

Keeping state synced correctly between peers and delegates is difficult. We must
pay close attention to leader lease expiration if we want to make change to
Raft.

# Alternatives

None that are immediately obvious. There must be some execution entity to handle
requests if we want to separate read requests from the raftstore.

# Unresolved questions

None.

[raft_cmdpb.proto]: https://github.com/pingcap/kvproto/blob/5e6e69a5ed381bd4a8afe7cb96cc47f955f6d160/proto/raft_cmdpb.proto
