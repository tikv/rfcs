# Coprocessor Plugin

## Summary

Add a general and pluggable coprocessor framework for RawKV mode.

## Motivation

TiKV is the storage component in the TiDB ecosystem, however, the distribution computation principle suggests that computation should be as close to the data source as possible. Therefore, TiKV has embedded a subset of the TiDB executor framework to push down some computation tasks when applicable.

But TiKV's capability should be far beyond that, as many distributed components can be built on top of TiKV, such as cache, full text search engine, graph database and NoSQL database. And same as TiDB, these product will also like to push down specific computation to TiKV, which requires the coprocessor to be customizable, aka pluggable.

For instance, a full-text seraching engine will persist the origin document and n-gram index on TiKV. It'll be a waste of resource if we read back and then update the index from a client. In contrary, the coprocessor plugin can generate the index from the origin document, and update the index inplace. What's more, the coprocessor plugin can perform index scan directly on TiKV.

The goals of the coprocessor plugin are:

- Do what client can do (on single region)
- Provide more guarantee than client does on RawKV
    - Raft transaction on RawKV
- Easy to use
    - Out of box
    - Easy to deploy
- Robust
    - Easy to debug
    - Log support, metrics support

## Detailed design

### Dynamic vs statically

Generally, there are two strategies to build a plugin framework: dynamically and statically, which means to load the plugin on startup or to embed in the binary on compilation.

![Plugin Arch](../media/plugin-arch.png)

They have both pros and cons:

| Static | Dynamic |
| -- | -- |
| ◯ High performance | X Relatively slower |
| ◯ Easy to deploy | X Complexify the deploy process |
| X Build the entire TiKV | ◯ Easy to build |
| X Build very slow | ◯ Build fast |
| X Hard to debug | ◯ Easy to debug |

In this RFC, we'll only focus on the dynamic plugin framework that works in RawKV mode.

### Plugin runtime

The plugin runtime is a new component settling in `tikv::server::service::kv::Service`. It loads the dylib plugin, dispatches coprocessor request to the plugin, and proxy the API calls from plugin to the [`Storage`](https://tikv.github.io/doc/tikv/storage/struct.Storage.html).

The path of the plugin should be specified in the config file and be loaded at TiKV startup.

### Plugin SDK

The plugin SDK is a standalone rust library that help setup the build process for the plugin.

### Multi-plugin

Currently TiKV has only one coprocessor `tidb_query`. However, without further work on statically linked plugin and txn mode support, we can't strip it from official release. So, multiple coprocessor has to be supported. Basically, we may need to add a `gPRC` rpc for coprocessor v2 request, in which coprocessor name and version is given, so that TiKV will be able to dispatch the request to the proper coprocessor, as well as to reject the request on version mismatch.

### Protobuf design

```proto
message RawCoprocessorRequest {
    kvrpcpb.Context context = 1;

    string copr_name = 2;
    string copr_version_constraint = 3;

    bytes data = 4;
}

message RawCoprocessorResponse {
    bytes data = 1;

    errorpb.Error region_error = 2;
    string other_error = 3;
}
```

### API design

To reduce the learning overhead, it'll be better that the API of the coprocessor plugin get closer to the client. Thus, it'll looks like the `RawClient` in the [Rust Client](https://github.com/tikv/client-rust) with extra txn-like methods e.g. `commit` and `lock`.

```rust
use std::ops::Range;

pub type Key = Vec<u8>;
pub type Value = Vec<u8>;
pub type KvPair = (Key, Value);

#[derive(Debug)]
pub struct Region {
    id: u64,
    start_key: Key,
    end_key: Key,
    region_epoch: RegionEpoch,
}

#[derive(Debug)]
pub struct RegionEpoch {
    pub conf_ver: u64,
    pub version: u64,
}

#[derive(Debug)]
pub enum Error {
    KeyNotInRegion { key: Key, region: Region },
    // More
}

pub type Result<T> = std::result::Result<T, Error>;

#[async_trait]
pub trait RawStorage: Send {
    async fn get(&self, key: Key) -> Result<Option<Value>>;
    async fn batch_get(&self, keys: Vec<Key>) -> Result<Vec<KvPair>>;
    async fn scan(&self, key_range: Range<Key>) -> Result<Vec<Value>>;
    async fn put(&mut self, key: Key, value: Value) -> Result<()>;
    async fn batch_put(&mut self, kv_pairs: Vec<KvPair>) -> Result<()>;
    async fn delete(&mut self, key: Key) -> Result<()>;
    async fn batch_delete(&mut self, keys: Vec<Key>) -> Result<()>;
    async fn delete_range(&mut self, key_range: Range<Key>) -> Result<()>;
}

pub trait Coprocessor: Send + Sync {
    fn on_raw_coprocessor_request(
        &self,
        region: Region,
        request: Vec<u8>,
        storage: Box<dyn RawStorage>,
    ) -> Result<Vec<u8>>;
```

### Keyspace

Keyspace[[RFC]](https://github.com/tikv/rfcs/pull/39)[[The most updated design doc]](https://docs.google.com/document/d/1x17-urAqToDo8TVXJroEHtc76fdssFaoANjSaNDhjKg/edit) is an incoming feature of TiKV that is highly related to coprocessor plugin. Keyspace determines whether a range of key should only be used in transaction mode or in RawKV mode. Since coprocessor works in either RawKV mode or txn mode, surely coprocessor plugin framework should aware of Keyspace. The details is TBD.

## Future work

### Official coprocessor

Provide an official coprocessor plugin which defines common primitives like `bool`, `f32`, `UTF8String`, `Array<T>` along with codec for them, moreover, implements simple scanning expressions or even aggregations.

Two reasons for that:

1. To provide an example showing how to build a coprocessor plugin for TiKV and guaranteed to be always up-to-date.

2. To comfort simple use cases to TiKV, so as to avoid forcing the users to develop a new plugin at the beginning.

### Transaction mode

Transaction mode is way more complicated than the RawKV mode. The current picture of txn plugin is an 'interactive' component with the client, which means, the client and the coprocessor will talk to each others during the request being processed. This is especially for resolving locks and coordinating the transaction among multiple coprocessors in multiple TiKV nodes.

### Static mode

Discussed in the section above.

### Migrate `tidb_query`

When static plugin and txn plugin is implemented, we can move the coprocessor specific to TiDB out of TiKV, and the repository of TiDB might be a good new home.
