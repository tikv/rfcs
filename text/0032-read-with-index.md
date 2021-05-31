# Follower Read With Applied Indices

## Summary

This RFC proposes an enhancement of getting snapshots on followers. The idea is
if the read request carries an applied index, the peer can get snapshot locally
without any communication with its leader as long as only it applys Raft logs
to the given index. It's useful to reduce latency if the cluster is deployed
across multiple data centers.

## Motivation

For clusters deployed across multiple data centers, the system latency mainly
depends on the network round-trip time (RTT) between data centers. For example,
suppose PDs are
deployed in Beijing, and TiKVs are deployed in Beijing and Xi'an (for high
availability). If a client which is near to Xian wants to read a Region, it
needs to get a transaction timestamp from PDs (in Beijing), and then sends
requests to TiKVs in Beijing or Xian. For the latter case, TiKVs in Xian will
send read index requests to their leaders (in Beijing) internally, which still
involves a RTT across data centers.

Therefore, if we can add some proxies in the major data center, and let it help TiDBs
in Xi'an to get the transaction timestamp and the applied indices of all target
Regions,
the read latency between data centers will be reduced from 2 RTT to 1 RTT.

## Detailed design

### Proxy

As described above, we need to add a proxy service in the major datacenter.
Considering that the proxy service is better to be highly available, we can put it
in TiKV instances. It's easy to register a new gRPC service in TiKV server.
After that, we get a highly available proxy service! The proxy has only one
method:

```protobuf
service TsAndReadIndexProxy {
    rpc GetTsAndReadIndex(Request) returns (Response) {}
}
```

The implementation of `GetTsAndReadIndex` gets a timestamp from PD first,
and then call `ReadIndex` of service `Tikv` to get applied indices of all
Regions that are carried in `Request`. If any retryable error occurs, the proxy
will retry it internally. Finally it will return `Response` to the RPC caller,
which contains a transaction timestamp and some applied indices.

### Coprocessor Requests

Add a field `applied_index` in `kvrpcpb.Context`:

```protobuf
message coprocessor.Request {
    kvrpcpb.Context context = 1;
    // omit other fields...

    uint64 applied_index = 5;
}
message kvrpcpb.Context {
    bool replica_read = 12;
    // After a region applys Raft logs to `applied_index`, we can get a
    // snapshot for the region even if the peer is follower.
    uint64 applied_index = 15;
}
```

A question is how clients can know `applied_index` for every Region? The
answer is RPC `ReadIndex` in service `Tikv`. It's already implemented, so we can
call it directly in clients.

### Get Snapshots With Applied Index

After TiKV receives a read request with `applied_index` in `Context`, it needs
to get a snapshot with the given `applied_index`. So we also need to add a new
field in `RaftRequestHeader`:

```protobuf
message RaftRequestHeader {
    // omit other fields...
    uint64 applied_index = 9;
}
```

With this implementation, TiKV can get a snapshot directly after it applys Raft
logs to `applied_index`.

## Drawbacks

## Alternatives

## Unresolved questions
