# Follower Read With Applied Indices

## Summary

This RFC proposes a improvement about getting snapshot on followers, which is
if the read request carries an applied index, the peer can get snapshot locally
wihtout any commucation with its leader, if only it has applied to the given
index. It's useful to reduce latency if the cluster is deployed in multi
datacenters.

## Motivation

For clusters deployed in multi datacenters, the system latency could mainly
depend on the network RTT between datacenters. For example, suppose PDs are
deployed in Beijing, and TiKVs are deployed in Beijing and Xian (for high
availability). If a client which is near to Xian wants to read a region, it
needs to get a transaction timestamp from PDs (in Beijing), and then sends
requests to TiKVs in Beijing or Xian. For the latter case, TiKVs in Xian will
send read index requests to their leaders (in Beijing) internally, which still
involves a RTT crossing datacenters.

So, If we can add some proxies in the major datacenter, and let it help TiDBs
in Xian to get transaction timestamp and applied indices of all target regions,
the read latency between multi datacenter will be reduced from 2 RTT to 1 RTT.

## Detailed design

### Proxy

As above described, we need to add a proxy service in the major datacenter.
Considering the proxy service is better to be high available, we can put it
into TiKV instances. It's easy to register a new gRPC service in TiKV server.
After that, we get a high available proxy service! The proxy has only one
method:

```protobuf
service TsAndReadIndexProxy {
    rpc GetTsAndReadIndex(Request) returns (Response) {}
}
```

The implementation of `GetTsAndReadIndex` will get a timestamp from PD first,
and then call `ReadIndex` of service `Tikv` to get applied indices of all
regions which is carried in `Request`. If any retryable error occurs, the proxy
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
    // After a region applys to `applied_index`, we can get a
    // snapshot for the region even if the peer is follower.
    uint64 applied_index = 15;
}
```

A question is how clients can knows `applied_index` for every region? The
answer is RPC `ReadIndex` in service `Tikv`. It's already ready, so we can
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

So TiKV can get a snapshot directly after it applys to `applied_index`.

## Drawbacks

## Alternatives

## Unresolved questions
