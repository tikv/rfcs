# Coprocessor Plugin

## Summary

Add a general and pluggable coprocessor framework for RawKV mode.

## Motivation

TiKV is the storage component in the TiDB ecosystem, however, the distribution computation principle suggests that computation should be as close to the data source as possible. Therefore, TiKV has embedded a subset of the TiDB executor framework to push down some computation tasks when applicable.

But TiKV's capability should be far beyond that, as many distributed components can be built on top of TiKV, such as cache, full text search engine, graph database and NoSQL database. And same as TiDB, these product will also like to push down specific computation to TiKV, which requires the coprocessor to be customizable, aka pluggable.

For instance, a full-text seraching engine will persist the origin document and n-gram index on TiKV. It'll be a waste of resource if we read back and then update the index from a client. On the contrary, the coprocessor plugin can generate the index from the origin document, and update the index inplace. What's more, the coprocessor plugin can perform index scan directly on TiKV.

The goals of the coprocessor plugin are:

- Do what client can do

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
    // Coprorcessor version constraint following SEMVER definition.
    string copr_version_req = 3;

    repeated KeyRange ranges = 4;
    bytes data = 5;
}

message RawCoprocessorResponse {
    errorpb.Error region_error = 1;
    // Error message for cases like if no coprocessor with a matching name is found
    // or on a version mismatch between plugin_api and the coprocessor.
    string error = 2;
    bytes data = 3;
}
```

### API design

```rust
use std::ops::Range;

/// A raw key in the storage.
pub type Key = Vec<u8>;
/// A raw value from the storage.
pub type Value = Vec<u8>;
/// A pair of a raw key and its value.
pub type KvPair = (Key, Value);

/// Raw bytes of the request payload from the client to the coprocessor.
pub type RawRequest = Vec<u8>;
/// The response from the coprocessor encoded as raw bytes that are sent back to the client.
pub type RawResponse = Vec<u8>;

/// A plugin that allows users to execute arbitrary code on TiKV nodes.
///
/// If you want to implement a custom coprocessor plugin for TiKV, your plugin needs to implement
/// the [`CoprocessorPlugin`] trait.
///
/// Plugins can run setup code in their constructor and teardown code by implementing
/// [`std::ops::Drop`].
pub trait CoprocessorPlugin: Send + Sync {
    /// Handles a request to the coprocessor.
    ///
    /// The data in the `request` parameter is exactly the same data that was passed with the
    /// `RawCoprocessorRequest` in the `data` field. Each plugin is responsible to properly decode
    /// the raw bytes by itself.
    /// The same is true for the return parameter of this function. Upon successful completion, the
    /// function should return a properly encoded result as raw bytes which is then sent back to
    /// the client.
    ///
    /// Most of the time, it's a good idea to use Protobuf for encoding/decoding, but in general you
    /// can also send raw bytes.
    ///
    /// Plugins can read and write data from the underlying [`RawStorage`] via the `storage`
    /// parameter.
    fn on_raw_coprocessor_request(
        &self,
        ranges: Vec<Range<Key>>,
        request: RawRequest,
        storage: &dyn RawStorage,
    ) -> PluginResult<RawResponse>;
}

/// Storage access for coprocessor plugins.
///
/// [`RawStorage`] allows coprocessor plugins to interact with TiKV storage on a low level.
///
/// Batch operations should be preferred due to their better performance.
#[async_trait(?Send)]
pub trait RawStorage {
    /// Retrieves the value for a given key from the storage on the current node.
    /// Returns [`Option::None`] if the key is not present in the database.
    async fn get(&self, key: Key) -> PluginResult<Option<Value>>;

    /// Same as [`RawStorage::get()`], but retrieves values for multiple keys at once.
    async fn batch_get(&self, keys: Vec<Key>) -> PluginResult<Vec<KvPair>>;

    /// Same as [`RawStorage::get()`], but accepts a `key_range` such that values for keys in
    /// `[key_range.start, key_range.end)` are retrieved.
    /// The upper bound of the `key_range` is exclusive.
    async fn scan(&self, key_range: Range<Key>) -> PluginResult<Vec<Value>>;

    /// Inserts a new key-value pair into the storage on the current node.
    async fn put(&self, key: Key, value: Value) -> PluginResult<()>;

    /// Same as [`RawStorage::put()`], but inserts multiple key-value pairs at once.
    async fn batch_put(&self, kv_pairs: Vec<KvPair>) -> PluginResult<()>;

    /// Deletes a key-value pair from the storage on the current node given a `key`.
    /// Returns [`Result::Ok]` if the key was successfully deleted.
    async fn delete(&self, key: Key) -> PluginResult<()>;

    /// Same as [`RawStorage::delete()`], but deletes multiple key-value pairs at once.
    async fn batch_delete(&self, keys: Vec<Key>) -> PluginResult<()>;

    /// Same as [`RawStorage::delete()`], but deletes multiple key-values pairs at once
    /// given a `key_range`. All records with keys in `[key_range.start, key_range.end)`
    /// will be deleted. The upper bound of the `key_range` is exclusive.
    async fn delete_range(&self, key_range: Range<Key>) -> PluginResult<()>;
}

/// Result returned by operations on [`RawStorage`].
pub type PluginResult<T> = std::result::Result<T, PluginError>;

/// Error returned by operations on [`RawStorage`].
///
/// If a plugin wants to return a custom error, e.g. an error in the business logic, the plugin should
/// return an appropriately encoded error in [`RawResponse`]; in other words, plugins are responsible
/// for their error handling by themselves.
#[derive(Debug)]
pub enum PluginError {
    KeyNotInRegion {
        key: Key,
        region_id: u64,
        start_key: Key,
        end_key: Key,
    },
    Timeout(Duration),
    Canceled,

    /// Errors that can not be handled by a coprocessor plugin but should instead be returned to the
    /// client.
    ///
    /// If such an error appears, plugins can run some cleanup code and return early from the
    /// request. The error will be passed to the client and the client might retry the request.
    Other(Box<dyn Any>),
}
```

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
