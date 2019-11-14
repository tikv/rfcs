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
needs to get a transaction timestamp from PDs at in Beijing, and then sends
requests to TiKVs in Beijing or Xian. For the latter case, TiKVs in Xian will
send read index requests to their leaders in Beijing internally, which still
involves a RTT crossing datacenters. However, If the client can get timestamp
and applied index for all target regions simultaneously, and then send read
request to TiKVs with the timestamp and index, with the feature TiKVs can save
one RTT, so the system latency can be reduced considerably.

## Detailed design

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

If we want to get snapshots on regions' followers, we need to send `ReadIndex`
requests to their leaders, or with this feature, wait for regions apply to
`applied_index`s carried in read requests. This wait mechanism can be easily
implemented with an obsrever.

## Drawbacks

The observer needs to be implemented carefully, otherwise performance issue
could be introduced. And, it's better to add an option to enable the feature.

## Alternatives

## Unresolved questions
