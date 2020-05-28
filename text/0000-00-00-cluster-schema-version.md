# Cluster Schema Version

## Summary

Add a schema version to TiKV clusters marking incompatible changes. The cluster
schema version is the lowest common schema version that all TiKV instances
support. TiKVs can know whether all instances have been upgraded to a compatible
version and when to start the new behavior.

## Motivation

To further improve our transactional model, we may need to change how we save
the metadata related to transactions. During a rolling update or after
upgrading a cluster, the running TiKV versions or the data format may be
inconsistent. A schema version of the cluster and regions can help us decide
when to switch to new behavior when the cluster becomes consistent again.

The cluster schema version is increased when all the TiKV instances in the
cluster support the new schema version. Once PD increases the cluster schema
version, it will reject any TiKV instance not supporting that schema version
joining in the cluster.

### Scenario 1: Add a new column family

If we need to save data in a new column family in a new version of TiKV, the
TiKV instances of the new version can handle raft commands that mutations.
However, during a rolling update, some instances are still using the old
version so they will fail to apply the commands writing data to a new column
family.

For example, we want to put all rollback records to a new column family. But we
must wait until all TiKV instances in the cluster are ready to apply commands
that write data to the new column family.

A cluster schema version can help us solve this problem. If the cluster schema
version is lower than a specific value, none of the instances in the cluster
will do operations of writing data to the new column family.

### Scenario 2: New format of column data

Currently, we have two column families "lock" and "write" to save metadata
related to transactions. In the future, we may change the format of the
metadata or add a new field to the metadata causing TiKV instances of old
versions to fail to parse them correctly.

Therefore, just like Scenario 1, before any instance in the cluster writes
the metadata in the new format, we must make sure the cluster schema version
is greater than a specific value.

## Detailed design

Every time we make a change that is incompatible with the older version, we
define a new version for it.

Each TiKV version defines the maximum schema version it supports. When TiKV
starts up, the version is passed through the `put_store` RPC.

```protobuf
message PutStoreRequest {
    ...
    uint64 max_supported_schema_version = 3;
}
```

If the version is less than the cluster schema version, PD returns an error with
`INCOMPATIBLE_VERSION`.

If the maximum supported schema version of all stores in the cluster is greater
than the cluster schema version, PD increases the cluster schema version. Later
TiKVs that do not support the increased version are rejected from joining in.

The updated cluster schema version is broadcast to all TiKV instances in the
response of the `store_heartbeat` RPC.

```protobuf
message StoreHeartbeatResponse {
    ...
    uint64 schema_version = 3;
}
```

The version is exposed to higher levels through a trait function of
`storage::kv::Engine`. It can be also included in the snapshot returned by
`Engine` for convenience.

```rust
pub trait Engine: Send + Clone + 'static {
    type Snap: Snapshot;

    ...

    fn get_cluster_schema_version(&self) -> u64;
}

pub trait Snapshot: Send + Clone {
    type Iter: Iterator;

    ...

    fn get_cluster_schema_version(&self) -> u64;
}
```

The transaction or MVCC layer should choose the right implementation according
to the cluster schema version.

## Drawbacks

Happy to hear feedback.

## Alternatives

An alternative is to reject rolling updates from a version that is not
compatible with the operations of the new version. For example, if TiKV `B.z`
has a behavior that is not compatible with TiKV `A.x`, it is not allowed to
directly upgrade TiKV `A.x` to TiKV `B.z`.

Instead, another version TiKV `A.y` which is compatible with the latest changes
in `B.z` must be released. To upgrade an `A.x` cluster to a `B.z` cluster, it
should upgrade to `A.y` first.

This requires the user an extra step to upgrade from a low version. It also
increases the risk that the user does not follow the version constraint. 

## Unresolved questions

None.

## Future possibilities

The similar schema version may be added to regions as well.

For example, we may add a new column family for latest data. With that column
family, usually we only need to consult the latest data so we can gain
performance improvement. But a cluster upgraded from an old version does not
have all latest data in the "latest" column family, making us not able to
benefit from the new column family.

In this case we can migrate the latest data in the original column family to the
"latest" column family. After the migration of a region, we can increase the
schema version of this region. The region schema version is also returned with
the snapshot. The logic built upon the raftstore can choose implementations
according to the schema version.