# TiKV Client (Rust)

## Summary

Introduce a full featured, official TiKV (and PD) Rust client. It is intended
to be used as a reference implementation, or to provide a C-compatible binding
for future clients.

## Motivation

Currently, users of TiKV must use [TiDB's Go
Client](https://github.com/pingcap/tidb/blob/master/store/tikv/client.go),
which is not well packaged or documented. We would like to ensure that users
can easily use TiKV and PD without needing to use TiDB.

We think this would help encourage community participation in the TiKV project
and associated libraries. During talks with several potential corporate users
we discovered that there was an interest in using TiKV to resolve concerns such
as caching and raw key-value stores.

## Detailed design

### Supported targets

We will track the `stable` channel of Rust, using the 2018 edition.

We will also support the most recent `nightly` version of Rust, but users
should not feel the need to reach for nightly unless they are already using it.

### Naming

The [Rust API
Guidelines](https://rust-lang-nursery.github.io/api-guidelines/naming.html) do
not prescribe any particular crate name convention.

We choose to name the crate `tikv-client`. Choosing to seperate `tikv` and
`client` helps potentially unfamiliar users to immediately understand the
intent of the package. `tikvclient`, while understandable, is not immediately
parsable by a human.

All structures and functions will otherwise follow the [Rust API
Guidelines](https://rust-lang-nursery.github.io/api-guidelines/), some of which
will be enforced by `clippy`.

### Installation

To utilize the client programmatically, users will be able to add the
`tikv-client` crate to their `Cargo.toml`'s dependencies. Then they must use
the crate with `use tikv_client;`. Unfortunately due to Rust’s naming
conventions this inconsistency is in place.

### Usage

#### Two types of APIs

TiKV provides two types of APIs for developers:

- *The Raw Key-Value API:* If your application scenario does not need
  distributed transactions or MVCC (Multi-Version Concurrency Control) and only
  needs to guarantee the atomicity towards one key, you can use the Raw
  Key-Value API.
- *The Transactional Key-Value API:* If your application scenario requires
  distributed ACID transactions and the atomicity of multiple keys within a
  transaction, you can use the Transactional Key-Value API.

Generally, the Raw Key-Value API has higher throughput and lower latency
compared to the Transactional Key-Value API. If distributed ACID transactions
are not required, Raw Key-Value API is preferred over Transactional Key-Value
API for better performance and ease of use. **Please be aware that these two
types of APIs are mutually exclusive. Users should __never__ mix and use these
two types of APIs on a __same__ TiKV cluster.**

The client provides two types of APIs in two separate modules for developers to
choose from.

#### The common data types

- `Key`: raw binary data
- `Value`: raw binary data
- `KvPair`: Key-value pair type
- `KvFuture`: A specialized Future type for TiKV client
- `Config`: Configuration for client

#### Raw Key-Value API Basic Usage

To use the Raw Key-Value API, take the following steps:

1. Create an instance of Config to specify endpoints of PD (Placement Driver)
   and optional security config

    ```rust
    let config = Config::new(vec!["127.0.0.1:3379"]).with_security(
        PathBuf::from("/path/to/ca.pem"),
        PathBuf::from("/path/to/client.pem"),
        PathBuf::from("/path/to/client-key.pem"),
    );
    ```

2. Create a Raw Key-Value client.

    ```rust
    let client = raw::Client::new(&config);
    ```

3. Call the Raw Key-Value client methods to access the data on TiKV. The Raw
    Key-Value API contains following methods

    ```rust
    impl Client {
        pub fn new(_config: &Config) -> Connect;
        pub fn get(&self, key: impl Into<Key>) -> Get;
        pub fn batch_get(&self,
            keys: impl IntoIterator<Item = impl Into<Key>>
        ) -> BatchGet;
        pub fn put(&self, key: impl Into<Key>, value: impl Into<Value>) -> Put;
        pub fn batch_put(&self,
            pairs: impl IntoIterator<Item = impl Into<KvPair>>
        ) -> BatchPut;
        pub fn delete(&self, key: impl Into<Key>) -> Delete;
        pub fn batch_delete(&self,
            keys: impl IntoIterator<Item = impl Into<Key>>
        ) -> BatchDelete;
        pub fn scan(&self, range: impl KeyRange, limit: u32) -> Scan;
        pub fn batch_scan<Ranges, Bounds>(&self,
            ranges: impl IntoIterator<Item = impl KeyRange>,
            each_limit: u32
        ) -> BatchScan where Ranges: AsRef<[Bounds]>, Bounds: RangeBounds<Key>;
        pub fn delete_range(&self, range: impl KeyRange) -> DeleteRange;
    }
    ```

#### Usage example of the Raw Key-Value API

```rust
use std::path::PathBuf;

use futures::future::Future;
use tikv_client::*;

fn main() {
    let config = Config::new(vec!["127.0.0.1:3379"]).with_security(
        PathBuf::from("/path/to/ca.pem"),
        PathBuf::from("/path/to/client.pem"),
        PathBuf::from("/path/to/client-key.pem"),
    );
    let raw = raw::Client::new(&config)
        .wait()
        .expect("Could not connect to tikv");

    let key: Key = b"Company".to_vec().into();
    let value: Value = b"PingCAP".to_vec().into();

    raw.put(key.clone(), value.clone())
        .cf("test_cf")
        .wait()
        .expect("Could not put kv pair to tikv");
    println!("Successfully put {:?}:{:?} to tikv", key, value);

    let value = raw
        .get(key.clone())
        .cf("test_cf")
        .wait()
        .expect("Could not get value");
    println!("Found val: {:?} for key: {:?}", value, key);

    raw.delete(key.clone())
        .cf("test_cf")
        .wait()
        .expect("Could not delete value");
    println!("Key: {:?} deleted", key);

    raw.get(key)
        .cf("test_cf")
        .wait()
        .expect_err("Get returned value for not existing key");

    let keys: Vec<Key> = vec![b"k1".to_vec().into(), b"k2".to_vec().into()];

    let values = raw
        .batch_get(keys.clone())
        .cf("test_cf")
        .wait()
        .expect("Could not get values");
    println!("Found values: {:?} for keys: {:?}", values, keys);

    let start: Key = b"k1".to_vec().into();
    let end: Key = b"k2".to_vec().into();
    raw.scan(start.clone()..end.clone(), 10)
        .cf("test_cf")
        .key_only()
        .wait()
        .expect("Could not scan");

    let ranges = vec![start.clone()..end.clone(), start..end];
    raw.batch_scan(ranges, 10)
        .cf("test_cf")
        .key_only()
        .wait()
        .expect("Could not batch scan");
}
```

The result is like:

```bash
Successfully put Key([67, 111, 109, 112, 97, 110, 121]):Value([80, 105, 110,
103, 67, 65, 80]) to tikv
Found val: Value([80, 105, 110, 103, 67, 65, 80]) for key: Key([67, 111, 109,
112, 97, 110, 121])
Key: Key([67, 111, 109, 112, 97, 110, 121]) deleted
```

Raw Key-Value client only supports the
GET/BATCH_GET/PUT/BATCH_PUT/DELETE/BATCH_DELETE/SCAN/BATCH_SCAN/DELETE_RANGE
commands. The Raw Key-Value client can be safely and concurrently accessed by
multiple threads. Therefore, for one process, one client is enough generally.

#### Transactional Key-Value API Basic Usage

The Transactional Key-Value API is more complicated than the Raw Key-Value API.
Some transaction related concepts are listed as follows.

- Client

    Like the Raw Key-Value client, a Client is client to a TiKV cluster.

- Snapshot

    A Snapshot is the state of the TiKV data at a particular point of time,
which provides some readonly methods. The multiple reads of the same Snapshot
is guaranteed consistent.

- Transaction

    Like Transaction in SQL, a Transaction in TiKV symbolizes a series of read
and write operations performed within the service. Internally, a Transaction
consists of a Snapshot for reads, and a buffer for all writes. The default
isolation level of a Transaction is Snapshot Isolation.

To use the Transactional Key-Value API, take the following steps:

1. Create an instance of Config to specify endpoints of PD (Placement Driver)
    and optional security config

    ```rust
    let config = Config::new(vec!["127.0.0.1:3379"]).with_security(
        PathBuf::from("/path/to/ca.pem"),
        PathBuf::from("/path/to/client.pem"),
        PathBuf::from("/path/to/client-key.pem"),
    );
    ```

2. Create a Transactional Key-Value client.

    ```rust
    let client = transaction::Client::new(&config);
    ```

3. Create a Transaction.

    ```rust
    let txn = client.begin();
    ```

4. (Optional) Modify isolation level of a Transaction.

    ``` rust
    #[derive(Copy, Clone, Eq, PartialEq, Debug)]
    pub enum IsolationLevel {
        SnapshotIsolation,
        ReadCommitted,
    }

    txn.set_isolation_level(IsolationLevel::ReadCommitted);
    ```

5. (Optional) Modify data using a Transaction.

    The lifecycle of a Transaction is: _begin → {get, set, delete, scan} →
{commit, rollback}_.

6. Call the Transactional Key-Value API's methods to access the data on TiKV.
   The Transactional Key-Value API contains the following most commonly used
   methods:

    ```rust
    impl Client {
        pub fn new(config: &Config) -> Connect;
        pub fn begin(&self) -> Transaction;
        pub fn begin_with_timestamp(&self, timestamp: Timestamp) -> Transaction;
        pub fn snapshot(&self) -> Snapshot;
        pub fn current_timestamp(&self) -> Timestamp;
    }

    impl Transaction {
        pub fn commit(self) -> Commit;
        pub fn rollback(self) -> Rollback;
        pub fn lock_keys(&mut self,\
            keys: impl IntoIterator<Item = impl Into<Key>>
        ) -> LockKeys;
        pub fn is_readonly(&self) -> bool;
        pub fn start_ts(&self) -> Timestamp;
        pub fn snapshot(&self) -> Snapshot;
        pub fn set_isolation_level(&mut self, level: IsolationLevel);
        pub fn get(&self, key: impl Into<Key>) -> Get;
        pub fn batch_get(&self,
            keys: impl IntoIterator<Item = impl Into<Key>>
        ) -> BatchGet;
        pub fn scan(&self, range: impl RangeBounds<Key>) -> Scanner;
        pub fn scan_reverse(&self, range: impl RangeBounds<Key>) -> Scanner;
        pub fn set(&mut self,
            key: impl Into<Key>,
            value: impl Into<Value>
        ) -> Set;
        pub fn delete(&mut self, key: impl Into<Key>) -> Delete;
    }

    impl Snapshot {
        pub fn get(&self, key: impl Into<Key>) -> Get;
        pub fn batch_get(&self,
            keys: impl IntoIterator<Item = impl Into<Key>>
        ) -> BatchGet;
        pub fn scan(&self, range: impl RangeBounds<Key>) -> Scanner;
        pub fn scan_reverse(&self, range: impl RangeBounds<Key>) -> Scanner;
    }

    ```

7. Complete the transaction.

    ```rust
    txn.commit();
    ```

#### Usage example of the Transactional Key-Value API

```rust
use std::ops::RangeBounds;
use std::path::PathBuf;

use futures::{future, Future, Stream};
use tikv_client::transaction::{Client, IsolationLevel};
use tikv_client::*;

fn puts(client: &Client, pairs: impl IntoIterator<Item = impl Into<KvPair>>) {
    let mut txn = client.begin();
    let _: Vec<()> = future::join_all(
        pairs
            .into_iter()
            .map(Into::into)
            .map(|p| txn.set(p.key().clone(), p.value().clone())),
    ).wait()
    .expect("Could not set key value pairs");
    txn.commit().wait().expect("Could not commit transaction");
}

fn get(client: &Client, key: Key) -> Value {
    let txn = client.begin();
    txn.get(key).wait().expect("Could not get value")
}

fn scan(client: &Client, range: impl RangeBounds<Key>, mut limit: usize) {
    client
        .begin()
        .scan(range)
        .take_while(move |_| {
            Ok(if limit == 0 {
                false
            } else {
                limit -= 1;
                true
            })
        }).for_each(|pair| {
            println!("{:?}", pair);
            Ok(())
        }).wait()
        .expect("Could not scan keys");
}

fn dels(client: &Client, keys: impl IntoIterator<Item = Key>) {
    let mut txn = client.begin();
    txn.set_isolation_level(IsolationLevel::ReadCommitted);
    let _: Vec<()> = keys
        .into_iter()
        .map(|p| {
            txn.delete(p).wait().expect("Could not delete key");
        }).collect();
    txn.commit().wait().expect("Could not commit transaction");
}

fn main() {
    let config = Config::new(vec!["127.0.0.1:3379"]).with_security(
        PathBuf::from("/path/to/ca.pem"),
        PathBuf::from("/path/to/client.pem"),
        PathBuf::from("/path/to/client-key.pem"),
    );
    let txn = Client::new(&config)
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
    let value1 = get(&txn, key1.clone());
    println!("{:?}", (key1, value1));

    // scan
    let key1: Key = b"key1".to_vec().into();
    scan(&txn, key1.., 10);

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

### Tooling

The `tikv_client` crate will be tested with Travis CI using Rust's standard
testing framework. We will also include benchmark with criterion in the future.
For public functions which process user input, we will seek to use fuzz testing
such as `proptest` to find subtle bugs.

The CI will validate all code is warning free, passes `rustfmt`, and passes a
`clippy` check without lint warnings.

All code that reaches the `master` branch should not output errors when the
following commands are run:

```shell
cargo fmt --all -- --check
cargo clippy --all -- -D clippy
cargo test --all -- --nocapture
cargo bench --all -- --test
```

## Drawbacks

Choosing not to create a Rust TiKV client would mean the current state of
clients remains the same.

It is likely that in the future we would end up creating a client in some other
form due to customer demand.

## Alternatives

### Package the Go client

Choosing to do this would likely be considerably much less work. The code is
already written, so most of the work would be documenting and packaging.
Unfortunately, Go does not share the same performance characteristics and FFI
capabilities as Rust, so it is a poor core binding for future libraries. Due to
the limited abstractions available in Go (it does not have a Linear Type
System) we may not be able to create the semantic abstractions possible in a
Rust client, reducing the quality of implementations referencing the client.

### Choose another language

We can choose another language such as C, C++, or Python.

A C client would be the most portable and allow future users and customers to
bind to the library as they wish. This quality is maintained in Rust, so it is
not an advantage for C. Choosing to implement this client in C or C++ means we
must take extra steps to support multiple packaging systems, string libraries,
and other choices which would not be needed in languages like Ruby, Node.js, or
Python.

Choosing to use Python or Ruby for this client would likely be considerably
less work than C/C++ as there is a reduced error surface. These languages do
not offer good FFI bindings, and often require starting up a language runtime.
We suspect that if we implement a C/C++/Rust client, future dynamic language
libraries will be able to bind to the Rust client, allowing them to be written
quickly and easily.

## Unresolved questions

There are some concerns about integration testing. While we can use the mock
`Pd` available in TiKV to mock PD, we do not currently have something similar
for TiKV. We suspect implementing a mock TiKV will be the easiest method.
