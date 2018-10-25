# Summary

Introduce a full featured, official TiKV (and PD) Rust client. It will be intended to be used as a reference implementation, or to provide C-compatible binding for future clients.

# Motivation

Currently, users of TiKV must use [TiDB's Go Client](https://github.com/pingcap/tidb/blob/master/store/tikv/client.go), which is not well packaged or documented. We would like to ensure that users can easily use TiKV and PD without needing to use TiDB.

We think this would help encourage community participation in the TiKV project and associated libraries. During talks with several potential corporate users we discovered that there was an interest in using TiKV to resolve concerns such as caching and raw key-value stores.

# Detailed design
## Supported Targets

We will target the `stable` channel of Rust starting in the Rust 2018 edition. We choose to begin with Rust 2018 so we do not need to concern ourselves with an upgrade path.

We will also support the most recent `nightly` version of Rust, but users should not feel the need to reach for stable unless they are already using it.

## Naming

While the [Rust API Guidelines](https://rust-lang-nursery.github.io/api-guidelines/naming.html) do not perscribe any particular crate name convention.

We choose to name the crate `tikv_client` to conform to the constraints presented to us by the Rust compiler. Cargo permits `tikv-client` and `tikv_client`, but `rustc` does not permit `tikv-client` so we choose to use `tikv_client` to reduce mental overhead.

Choosing to seperate `tikv` and `client` helps potentially unfamiliar users to immediately understand the intent of the package. `tikvclient`, while understandable, is not immediately parsable by a human.

All structures and functions will otherwise follow the [Rust API Guidelines](https://rust-lang-nursery.github.io/api-guidelines/), some of which will be enforced by `clippy`.

## Installation.

To utilize the client programmatically, users will be able to add the `tikv-client` crate to their `Cargo.toml`'s dependencies. Then they must use the crate with `use tikv_client;`. Unfortunately due to Rust’s naming conventions this inconsistency is in place.

To utilize the command line client, users will be able to install the binary via `cargo install tikv-client`. They will then be able to access the client through the binary `tikv-client`. If they wish for a different name they can alias it in their shell.

## Usage

## Key structs and traits
Raw Kv Client
```rust
Client {
    fn new(config: &Config) -> KvFuture<Self>;
    fn get<K, C>(&self, key: K, cf: C) -> KvFuture<Value>
    where
        K: Into<Key>,
        C: Into<Option<String>>;
    fn batch_get<I, K, C>(&self, keys: I, cf: C) -> KvFuture<Vec<KvPair>>
    where
        I: IntoIterator<Item = K>,
        K: Into<Key>,
        C: Into<Option<String>>;
    fn put<P, C>(&self, pair: P, cf: C) -> KvFuture<()>
    where
        P: Into<KvPair>,
        C: Into<Option<String>>;
    fn batch_put<I, P, C>(&self, pairs: I, cf: C) -> KvFuture<()>
    where
        I: IntoIterator<Item = P>,
        P: Into<KvPair>,
        C: Into<Option<String>>;
    fn delete<K, C>(&self, key: K, cf: C) -> KvFuture<()>
    where
        K: Into<Key>,
        C: Into<Option<String>>;
    fn batch_delete<I, K, C>(&self, keys: I, cf: C) -> KvFuture<()>
    where
        I: IntoIterator<Item = K>,
        K: Into<Key>,
        C: Into<Option<String>>;
    fn scan<R, C>(&self, range: R, limit: u32, key_only: bool, cf: C) -> KvFuture<Vec<KvPair>>
    where
        R: Into<KeyRange>,
        C: Into<Option<String>>;
    fn batch_scan<I, R, C>(
        &self,
        ranges: I,
        each_limit: u32,
        key_only: bool,
        cf: C,
    ) -> KvFuture<Vec<KvPair>>
    where
        I: IntoIterator<Item = R>,
        R: Into<KeyRange>,
        C: Into<Option<String>>;
    fn delete_range<R, C>(&self, range: R, cf: C) -> KvFuture<()>
    where
        R: Into<KeyRange>,
        C: Into<Option<String>>;
}
```
Transactional Kv Client
```rust
Client {
    fn new(config: &Config) -> KvFuture<Self>；
    fn begin(&self) -> KvFuture<Transaction>;
    fn begin_with_timestamp(&self, _timestamp: Timestamp) -> KvFuture<Transaction>;
    fn snapshot(&self) -> KvFuture<Snapshot>;
    fn current_timestamp(&self) -> Timestamp;
}

Transaction {
    fn commit(&mut self) -> KvFuture<()>;
    fn rollback(&mut self) -> KvFuture<()>;
    fn lock_keys<I, K>(&mut self, keys: I) -> KvFuture<()>
    where
        I: IntoIterator<Item = I>,
        K: Into<Key>;
    fn is_readonly(&self) -> bool;
    fn start_ts(&self) -> Timestamp;
    fn snapshot(&self) -> KvFuture<Snapshot>;
    fn get<K>(&self, key: K) -> KvFuture<Value>
    where
        K: Into<Key>;
    fn batch_get<I, K>(&self, keys: I) -> KvFuture<Vec<KvPair>>
    where
        I: IntoIterator<Item = K>,
        K: Into<Key>;
    fn seek<K>(&self, key: K) -> KvFuture<Scanner>
    where
        K: Into<Key>;
    fn seek_reverse<K>(&self, key: K) -> KvFuture<Scanner>
    where
        K: Into<Key>;
    fn set<P>(&mut self, pair: P) -> KvFuture<()>
    where
        P: Into<KvPair>;
    fn delete<K>(&mut self, key: K) -> KvFuture<()>
    where
        K: Into<Key>;
}

Snapshot {
    fn get<K>(&self, key: K) -> KvFuture<Value>
    where
        K: Into<Key>;
    fn batch_get<I, K>(&self, keys: I) -> KvFuture<Vec<KvPair>>
    where
        I: IntoIterator<Item = K>,
        K: Into<Key>;
    fn seek<K>(&self, key: K) -> KvFuture<Scanner>
    where
        K: Into<Key>;
    fn seek_reverse<K>(&self, key: K) -> KvFuture<Scanner>
    where
        K: Into<Key>;
}

```

## Programming model

The client instance is thread safe and all the interfaces return futures so users can use the client in either asynchronous or synchronous way. A dedicated event loop thread is created at per client instance basis to drive the reactor to make progress.

## Tooling

The `tikv_client` crate will be tested with Travis CI using Rust's standard testing framework. We will also include benchmarking with criterion in the future. For public functions which process user input, we will seek to use fuzz testing such as `quickcheck` to find subtle bugs.

The CI will validate all code is warning free, passes `rustfmt`, and passes a `clippy` check without lint warnings.

All code that reaches the `master` branch should not output errors when the following commands are run:

```shell
cargo fmt --all -- --check
cargo clippy --all -- -D clippy
cargo test --all -- --nocapture
cargo bench --all -- --test
```

# Drawbacks

Choosing not to create a Rust TiKV client would mean the current state of clients remains the same.

It is likely that in the future we would end up creating a client in some other form due to customer demand.

# Alternatives

## Package the Go client

Choosing to do this would likely be considerably much less work. The code is already written, so most of the work would be documenting and packaging. Unfortunately, Go does not share the same performance characteristics and FFI capabilities as Rust, so it is a poor core binding for future libraries. Due to the limited abstractions available in Go (it does not have a Linear Type System) we may not be able to create the semantic abstractions possible in a Rust client, reducing the quality of implementations referencing the client.

## Choose another language

We can choose another language such as C, C++, or Python.

A C client would be the most portable and allow future users and customers to bind to the library as they wish. This quality is maintained in Rust, so it is not an advantage for C. Choosing to implement this client in C or C++ means we must take extra steps to support multiple packaging systems, string libraries, and other choices which would not be needed in languages like Ruby, Node.js, or Python.

Choosing to use Python or Ruby for this client would likely be considerably less work than C/C++ as there is a reduced error surface. These languages do not offer good FFI bindings, and often require starting up a language runtime. We suspect that if we implement a C/C++/Rust client, future dynamic language libraries will be able to bind to the Rust client, allowing them to be written quickly and easily.

# Unresolved questions

There are some concerns about integration testing. While we can use the mock `Pd` available in TiKV to mock PD, we do not currently have something similar for TiKV. We suspect implementing a mock TiKV will be the easiest method.

