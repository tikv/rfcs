# Summary

Introduce a full featured, official TiKV (and PD) Rust client. It is intended to be used as a reference implementation, or to provide C-compatible binding for future clients.

# Motivation

Currently, users of TiKV must use [TiDB's Go Client](https://github.com/pingcap/tidb/blob/master/store/tikv/client.go), which is not well packaged or documented. We would like to ensure that users can easily use TiKV and PD without needing to use TiDB.

We think this would help encourage community participation in the TiKV project and associated libraries. During talks with several potential corporate users we discovered that there was an interest in using TiKV to resolve concerns such as caching and raw key-value stores.

# Detailed design
## Supported targets

We will target the `stable` channel of Rust starting in the Rust 2018 edition. We choose to begin with Rust 2018 so we do not need to concern ourselves with an upgrade path.

We will also support the most recent `nightly` version of Rust, but users should not feel the need to reach for nightly unless they are already using it.

## Naming

The [Rust API Guidelines](https://rust-lang-nursery.github.io/api-guidelines/naming.html) do not perscribe any particular crate name convention.

We choose to name the crate `tikv_client` to conform to the constraints presented to us by the Rust compiler. Cargo permits `tikv-client` and `tikv_client`, but `rustc` does not permit `tikv-client` so we choose to use `tikv_client` to reduce mental overhead.

Choosing to seperate `tikv` and `client` helps potentially unfamiliar users to immediately understand the intent of the package. `tikvclient`, while understandable, is not immediately parsable by a human.

All structures and functions will otherwise follow the [Rust API Guidelines](https://rust-lang-nursery.github.io/api-guidelines/), some of which will be enforced by `clippy`.

## Installation

To utilize the client programmatically, users will be able to add the `tikv-client` crate to their `Cargo.toml`'s dependencies. Then they must use the crate with `use tikv_client;`. Unfortunately due to Rust’s naming conventions this inconsistency is in place.

To utilize the command line client, users will be able to install the binary via `cargo install tikv-client`. They will then be able to access the client through the binary `tikv-client`. If they wish for a different name they can alias it in their shell.

## Usage

## Two types of APIs
TiKV provides two types of APIs for developers:
- The Raw Key-Value API

    If your application scenario does not need distributed transactions or MVCC (Multi-Version Concurrency Control) and only needs to guarantee the atomicity towards one key, you can use the Raw Key-Value API.

- The Transactional Key-Value API

    If your application scenario requires distributed ACID transactions and the atomicity of multiple keys within a transaction, you can use the Transactional Key-Value API.

Generally, the Raw Key-Value API has higher throughput and lower latency compared to the Transactional Key-Value API. If distributed ACID transactions are not required, Raw Key-Value API is preferred over Transactional Key-Value API for better performance and ease of use.

The client provides two types of APIs in two separate modules for developers to choose from.

### The common data types

- Key: raw binary data
- Value: raw binary data
- KvPair: Key-value pair type
- KeyRange: Half-open interval of keys
- Config: Configuration for client

### Raw Key-Value API Basic Usage

To use the Raw Key-Value API, take the following steps:

1. Create an instance of Config to specify endpoints of PD (Placement Driver) and optional security config

    ```rust
    let config = Config::new(vec!["127.0.0.1:2379"]);
    ```

2. Create a Raw Key-Value client.

    ```rust
    let client = RawClient::new(&config);
    ```

3. Call the Raw Key-Value client methods to access the data on TiKV. The Raw Key-Value API contains following methods

    ```rust
    pub trait Client {
        type AClient: Client;

        fn get(&self, key: impl AsRef<Key>) -> Get<Self::AClient>;

        fn batch_get(&self, keys: impl AsRef<[Key]>) -> BatchGet<Self::AClient>;

        fn put(&self, pair: impl Into<KvPair>) -> Put<Self::AClient>;

        fn batch_put(
            &self,
            pairs: impl IntoIterator<Item = impl Into<KvPair>>,
        ) -> BatchPut<Self::AClient>;

        fn delete(&self, key: impl AsRef<Key>) -> Delete<Self::AClient>;

        fn batch_delete(&self, keys: impl AsRef<[Key]>) -> BatchDelete<Self::AClient>;

        fn scan(&self, range: impl RangeBounds<Key>, limit: u32) -> Scan<Self::AClient>;

        fn batch_scan<Ranges, Bounds>(
            &self,
            ranges: Ranges,
            each_limit: u32,
        ) -> BatchScan<Self::AClient>
        where
            Ranges: AsRef<[Bounds]>,
            Bounds: RangeBounds<Key>;

        fn delete_range(&self, range: impl RangeBounds<Key>) -> DeleteRange<Self::AClient>;
    }
    ```

#### Usage example of the Raw Key-Value API

    ```rust
    extern crate futures;
    extern crate tikv_client;

    use futures::future::Future;
    use tikv_client::raw::Client;
    use tikv_client::*;

    fn main() {
        let config = Config::new(vec!["127.0.0.1:3379"]);
        let raw = raw::RawClient::new(&config)
            .wait()
            .expect("Could not connect to tikv");

        let key: Key = b"Company".to_vec().into();
        let value: Value = b"PingCAP".to_vec().into();

        raw.put((Clone::clone(&key), Clone::clone(&value)))
            .cf("test_cf")
            .wait()
            .expect("Could not put kv pair to tikv");
        println!("Successfully put {:?}:{:?} to tikv", key, value);

        let value = raw
            .get(&key)
            .cf("test_cf")
            .wait()
            .expect("Could not get value");
        println!("Found val: {:?} for key: {:?}", value, key);

        raw.delete(&key)
            .cf("test_cf")
            .wait()
            .expect("Could not delete value");
        println!("Key: {:?} deleted", key);

        raw.get(&key)
            .cf("test_cf")
            .wait()
            .expect_err("Get returned value for not existing key");

        let keys = vec![b"k1".to_vec().into(), b"k2".to_vec().into()];

        let _values = raw
            .batch_get(&keys)
            .cf("test_cf")
            .wait()
            .expect("Could not get values");

        let start: Key = b"k1".to_vec().into();
        let end: Key = b"k2".to_vec().into();
        raw.scan(&start..&end, 10)
            .cf("test_cf")
            .key_only()
            .wait()
            .expect("Could not scan");

        let ranges = [&start..&end, &start..&end];
        raw.batch_scan(&ranges, 10)
            .cf("test_cf")
            .key_only()
            .wait()
            .expect("Could not batch scan");
    }
    ```

The result is like:

    ```bash
    Successfully put Key([67, 111, 109, 112, 97, 110, 121]):Value([80, 105, 110, 103, 67, 65, 80]) to tikv
    Found val: Value([80, 105, 110, 103, 67, 65, 80]) for key: Key([67, 111, 109, 112, 97, 110, 121])
    Key: Key([67, 111, 109, 112, 97, 110, 121]) deleted
    ```

Raw Key-Value client is a client of the TiKV server and only supports the GET/BATCH_GET/PUT/BATCH_PUT/DELETE/BATCH_DELETE/SCAN/BATCH_SCAN/DELETE_RANGE commands. The Raw Key-Value client can be safely and concurrently accessed by multiple threads. Therefore, for one process, one client is enough generally.

### Try the Transactional Key-Value API

The Transactional Key-Value API is more complicated than the Raw Key-Value API. Some transaction related concepts are listed as follows.

- Client

    Like the Raw Key-Value client, a Client is client to a TiKV cluster.

- Snapshot

    A Snapshot is the state of a Client at a particular point of time, which provides some readonly methods. The multiple reads of the same Snapshot is guaranteed consistent.

- Transaction

    Like the Transaction in SQL, a Transaction symbolizes a series of read and write operations performed within the Client. Internally, a Transaction consists of a Snapshot for reads, and a buffer for all writes. The default isolation level of a Transaction is Snapshot Isolation.

To use the Transactional Key-Value API, take the following steps:

1. Create an instance of Config to specify endpoints of PD (Placement Driver) and optional security config

    ```rust
    let config = Config::new(vec!["127.0.0.1:2379"]);
    ```

2. Create a Transactional Key-Value client.

    ```rust
    let client = TxnClient::new(&config);
    ```

3. (Optional) Modify data using a Transaction.

    The lifecycle of a Transaction is: _begin → {get, set, delete, scan} → {commit, rollback}_.

4. Call the Transactional Key-Value API's methods to access the data on TiKV. The Transactional Key-Value API contains the following methods:

    ```rust
    fn begin(&self) -> KvFuture<Transaction>;

    fn get<K>(&self, key: K) -> KvFuture<Value>
    where
        K: AsRef<Key>;

    fn set<P>(&mut self, pair: P) -> KvFuture<()>
    where
        P: Into<KvPair>;

    fn delete<K>(&mut self, key: K) -> KvFuture<()>
    where
        K: AsRef<Key>;

    fn seek<K>(&self, key: K) -> KvFuture<Scanner>
    where
        K: AsRef<Key>;

    fn commit(self) -> KvFuture<()>;

    fn rollback(self) -> KvFuture<()>;
    ```

### Usage example of the Transactional Key-Value API

    ```rust
    extern crate futures;
    extern crate tikv_client;

    use futures::{Async, Future, Stream};
    use tikv_client::transaction::{Client, Mutator, Retriever, TxnClient};
    use tikv_client::*;

    fn puts<P, I>(client: &TxnClient, pairs: P)
    where
        P: IntoIterator<Item = I>,
        I: Into<KvPair>,
    {
        let mut txn = client.begin().wait().expect("Could not begin transaction");
        let _: Vec<()> = pairs
            .into_iter()
            .map(Into::into)
            .map(|p| {
                txn.set(p).wait().expect("Could not set key value pair");
            }).collect();
        txn.commit().wait().expect("Could not commit transaction");
    }

    fn get(client: &TxnClient, key: &Key) -> Value {
        let txn = client.begin().wait().expect("Could not begin transaction");
        txn.get(key).wait().expect("Could not get value")
    }

    fn scan(client: &TxnClient, start: &Key, limit: usize) {
        let txn = client.begin().wait().expect("Could not begin transaction");
        let mut scanner = txn.seek(start).wait().expect("Could not seek to start key");
        let mut limit = limit;
        loop {
            if limit == 0 {
                break;
            }
            match scanner.poll() {
                Ok(Async::Ready(None)) => return,
                Ok(Async::Ready(Some(pair))) => {
                    limit -= 1;
                    println!("{:?}", pair);
                }
                _ => break,
            }
        }
    }

    fn dels<P>(client: &TxnClient, pairs: P)
    where
        P: IntoIterator<Item = Key>,
    {
        let mut txn = client.begin().wait().expect("Could not begin transaction");
        let _: Vec<()> = pairs
            .into_iter()
            .map(|p| {
                txn.delete(p).wait().expect("Could not delete key");
            }).collect();
        txn.commit().wait().expect("Could not commit transaction");
    }

    fn main() {
        let config = Config::new(vec!["127.0.0.1:3379"]);
        let txn = TxnClient::new(&config)
            .wait()
            .expect("Could not connect to tikv");

        // set
        let key1: Key = b"key1".to_vec().into();
        let value1: Value = b"value1".to_vec().into();
        let key2: Key = b"key2".to_vec().into();
        let value2: Value = b"value2".to_vec().into();
        puts(&txn, vec![(key1, value1), (key2, value2)]);

        // get
        let key1: Key = b"key1".to_vec().into();
        let value1 = get(&txn, &key1);
        println!("{:?}", (key1, value1));

        // scan
        let key1: Key = b"key1".to_vec().into();
        scan(&txn, &key1, 10);

        // delete
        let key1: Key = b"key1".to_vec().into();
        let key2: Key = b"key2".to_vec().into();
        dels(&txn, vec![key1, key2]);
    }
    ```

The result is like:

    ```bash
    (Key([107, 101, 121, 49]), Value([118, 97, 108, 117, 101, 49]))
    (Key([107, 101, 121, 49]), Value([118, 97, 108, 117, 101, 49]))
    (Key([107, 101, 121, 49]), Value([118, 97, 108, 117, 101, 49]))
    ```

## Programming model

The client instance is thread safe and all the interfaces return futures so users can use the client in either asynchronous or synchronous way. A dedicated event loop thread is created at a per client instance basis to drive the reactor to make progress.

## Tooling

The `tikv_client` crate will be tested with Travis CI using Rust's standard testing framework. We will also include benchmark with criterion in the future. For public functions which process user input, we will seek to use fuzz testing such as `quickcheck` to find subtle bugs.

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

