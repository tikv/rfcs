# Summary
Abstract and design new API of key-value store engine for TiKV.

# Motivation

Currently, TiKV relies heavily on the API of [RocksDB](https://github.com/facebook/rocksdb). We want to propose common interfaces for key-value store engine, so that any data structure, even based on `HashMap`, which implements those interfaces could be used to provide low level key-value services for TiKV.

# Detailed design

Now, in TiKV, the functional modules being used are like:

* Open/Close database
* Read/Write/Delete
* Iteration
* Snapshot(required by `raftstore`)
* Atomic Updates
* ColumnFamily
* Key/File Compaction
* Monitor & Statistic
* TableProperties, UserCollectedProperties
* Modify/Ingest External Files
* Env
* Event Listener
* WAL

All APIs are separated into two parts: common ones and specific ones.

## Common API

```
Status
```
Returned by most functions which may encounter errors. Member functions like `bool ok()` and `string ToString()` should be provided to judge correctness.

```
Status Open(const &OpenOptions, OpenResults*)
```
Open database with options which contains information about `Database Options`, `ColumnFamily`, etc. 

```
Status Close(const CloseOptions&);
```
Close database with options.

```
Status Put(const PutOptions&, PutResult*)
```
Put data into database with options which contains information about `ColumnFamily`, `Key-Value Pair`, `IsSync`, etc.

```
Status Get(const GetOptions&, GetResult*);
```
Get data from database with options which contains information about `ColumnFamily`, `Snapshot`, `Prefix`, etc.

```
Status Iterator(const IteratorOptions&, IteratorResult*);
```
Options for `Iterator` should contain the information like `GetOptions` and some others such as `Lower Bound`, `Upper Bound`, `Iter Start Seqnum`, etc. The returned iterator should support functions like `Seek`, `Next`, `Prev`, `key`, `value`, etc.

```
Status Delete(const DeleteOptions&, DeleteResult*);
```
Delete data from database with options similar to `PutOptions`. `DeleteOptions` needs information like `Begin Key` and `End Key` to support `Range Delete`.

```
Status Write(const WriteOptions&, WriteResult*);
```
Batch write data into database. Options is similar to `PutOptions` but need information for `Batch Write` operation.

```
Status Statistics(const StatisticsOptions&, StatisticsInterface*);
```
Return the statistic interface of database. TiKV can get data from `StatisticsInterface` by FFI or `Specific Protocol`. Some metrics such as `ApproximateSizes` could be integrated into this.

```
Status SetConfig(const SetConfigOptions&, SetConfigResult*);
```
Dynamic set configuration. The data about configuration could be `Binary Flow` or `String`. 

## Specific API
```
Status CreateColumnFamily(const CreateColumnFamilyOptions&, CreateColumnFamilyResult*);
Status GetColumnFamily(const GetColumnFamilyOptions&, GetColumnFamilyResult*);
Status DropColumnFamily(const DropColumnFamilyOptions&, DropColumnFamilyResult*);
```
Interfaces for accessing `ColumnFamily`. 

```
Status Snapshot(const SnapshotOptions&, SnapshotResult*);
Status ReleaseSnapshot(const ReleaseSnapshotOptions&, ReleaseSnapshotResult*);
```
`Snapshot` is strongly required in `RaftStore` and is not easy to be replaced. Any database hasn't support this couldn't be used in `RaftStore` for now.

```
Status Property(const PropertyOptions&, PropertyResult*);
```
In RocksDB, there exists a concept of `Table` which usually can be regarded as `File`. Two kinds of properties are used in TiKV: `TableProperties` and `UserCollectedProperties`. So, this interface should be compatible with them.

```
Status ModifyExternalFile(const ModifyExternalFileOptions&, ModifyExternalFileResult*);
```
Modify external file. The module `snap` in TiKV needs this function to set `global_seq_no`, which got from `TableReader::GetTableProperties()`, into external sst file.

```
Status IngestExternalFile(const IngestExternalFileOptions&, IngestExternalFileResult*);
```
Ingest external file. `IngestExternalFileOptions` should contain information about `ColumnFamily`, `File Paths`, etc.

```
Status Compact(const CompactOptions&, CompactResult*);
```
Compact keys in range. Information like `ColumnFamily`, `Key Range`, `Compaction Level` should be contained in `CompactOptions`.

```
Status CompactFiles(const CompactFilesOptions&, CompactFilesResult*);
```
Compact files in range. Information like `ColumnFamily`, `File Range`, `Compaction Type` should be contained in `CompactFilesOptions`.

```
Status DeleteFiles(const DeleteFilesOptions&, DeleteFilesResult*);
```
Delete files in range. `ColumnFamily` and `File Range` should be contained in `DeleteFilesOptions`.

```
class EventListener;
```
Just like RocksDB, `OnFlushCompleted`, `OnCompactionCompleted`, `OnExternalFileIngested` and `OnStallConditionsChanged` are the basic monitored events. The instance of `EventListener` should be owned by `ColumnFamily`.

```
Status Flush(const FlushOptions&, FlushResult*);
```
Flush memtable into sstable. `IsSync` and `ColumnFamily` are needed in `FlushOptions`.

```
Status GetEnv(GetEnvResult*);
```
Get information of enviroment.

```
Status SyncWAL();
```
Sync the WAL.

## Implementation

- `ColumnFamily` is the basic partition mode of data. It consists of `Table`, which represents a cluster of key-value data in memory or file system.
- Main interfaces should be implemented by both `C++` and `Rust`. The layer of `C++` should be exposed to other databses becasue most of them are implemented by `C++`. The layer of `Rust` can interact with `C++` through `C Foreign Function Interface`. 
- The abstraction ability of `Rust` is not as good as `C++`. So we need to make the style of `C++` code compatible with `Rust Traits` by using `Template`.
- Many details need to be reconsidered during implementation.

# Drawbacks

- Code may become complicated.

# Alternatives

- None

# Unresolved questions
- Most specific APIs reference RocksDB but a few concepts like `Snapshot` and `WAL` are hard to be abstracted as common ones.
