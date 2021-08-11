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

## Document-level design

You can develop a plugin using Rust which have full read and write access to the data in TiKV node when you send a `RawCoprocessor` request from the client. The client and the plugin communitcates in raw bytes so you can build your own protocol on top of it.

To develop a plugin, intialized a Rust library project, and, in `Cargo.toml`, add the following lines:

```toml
[lib]
crate-type = ["dylib"]

[dependencies]
coprocessor_plugin_api = { git = "https://github.com/tikv/tikv.git" }
```

Then, in `src/lib.rs`, implement the following:

```rust
use coprocessor_plugin_api::*;
use std::ops::Range;

#[derive(Default)]
struct ExamplePlugin;

impl CoprocessorPlugin for ExamplePlugin {
    fn on_raw_coprocessor_request(
        &self,
        ranges: Vec<Range<Key>>,
        request: Vec<u8>,
        storage: &dyn RawStorage,
    ) -> PluginResult<Vec<u8>> {
        unimplemented!()
    }
}

declare_plugin!(ExamplePlugin::default());
```

`on_raw_coprocessor_request` will be invoked when you call `RawClient::coprocessor` on the client:

```rust
impl RawClient {
  pub async fn coprocessor(
    &self,
        copr_name: String,
        copr_version_req: String,
        ranges: impl IntoIterator<Item = impl Into<BoundRange>>,
        request_builder: impl Fn(Vec<Range<Key>>, Region) -> Vec<u8> + Send + Sync + 'static,
    ) -> Result<Vec<(Vec<u8>, Vec<Range<Key>>)>>;
}
```

The request will send to plugin with exact same name and matchable version of `SEMVER 2.0`. The name and version of the coprocessor plugin is inherited from the plugin's library name and version in `Cargo.toml`, otherwise, you can explictly specify the name and version in `declare_plugin!()`:

```rust
declare_plugin!("example_plugin", "0.1.0", ExamplePlugin::default());
```

You should specify the key range you will like to read or write and then the client will shard the key range by region and generate request to every single region by the builder function provided by user. The client will rebuild the request on region retry, e.g. network unstable, region split or region merge, so the plugin must be idempotent on every single key. Note that a request to a key might replay on different server (e.g. leader changed) within different key range.

In `on_raw_coprocessor_request` plugin will get the reference to `&dyn RawStorage`, by which you can access the raw data in the region:

```rust
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
```

Build the plugin **using the same `rustc` as TiKV are built** and then you will find the dylib in `target/debug` or `target/release`. Next, copy the `.so` (Linux) or `.dylib` (MacOS) file to the `coprocessors/` folder next to `tikv-server`, and then TiKV will load it automatically on startup or even while running. The `coprocessors/` folder is configurable by TiKV config file:

```toml
[coprocessor-v2]
## Path to the directory where compiled coprocessor plugins are located.
## Plugins in this directory will be automatically loaded by TiKV.
coprocessor-plugin-directory = "./coprocessors"
```

### Multi-plugin

Multiple plugin can be load at the same time as long as the plugins have different names or versions.

### Auto loading

TiKV will watch the plugin directory and load new plugin when new plugin file is created. New plugin should have new file name and should not have the same plugin name and version number as the already loaded plugin.

## Reference-level design

### Plugin runtime

The plugin runtime is a new component settling in `tikv::server::service::kv::Service`. It loads the dylib plugin, dispatches coprocessor request to the plugin, and proxy the API calls from plugin to the [`Storage`](https://tikv.github.io/doc/tikv/storage/struct.Storage.html).

The path of the plugin should be specified in the config file and be loaded at TiKV startup.

### Auto Loading

It should be convenient to be able to hot reload the plugin without restarting TiKV. But due to the dylib constraints described in [`rust-hot-reload`](https://github.com/irh/rust-hot-reloading#avoiding-thread-local-storage):

> Any use of TLS will prevent hot-reloading from working, even with the unique copy trick.

The hot-reload plugin should have new filename to avoid conflict with the existing plugins, which means the plugins will never be unload by TiKV, but you can register a new plugin with new version number or new name.

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

### Plugin API design

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

### Alloaction

Allacation on plugin frameworks is nasty problem for a long time now. We won't talk about other alternatives but focus on the solution we finally choose. The problem is that how the objects (e.g. `String`) alloacted by the plugin should be freed after being passed to the host, or vise versa. Thanks that the plugin and TiKV are both built in Rust, we can share the same allcator for them to solve this problem. In practice, TiKV will pass its function pointer `fn alloc()` `fn dealloc()` to the plugin, therefore the plugin can proxy its `fn alloc()` `fn dealloc()` to the host.

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
