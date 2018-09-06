# Summary

Support RPC `UnsafeDestroyRange`. This call is on the whole TiKV rather than a certain Region. When it is invoked, TiKV will use `delete_files_in_range` to quickly free a large amount of space, and then scan and delete all remaining keys in the range. Raft layer will be bypassed, and the range can cover multiple Regions. The invoker should promise that after invoking `UnsafeDestroyRange`, the range will never be accessed again. That is to say, the range is permanently scrapped.

This interface is only designed for TiDB. It's used to clean up data after dropping/truncating a huge table/index.

# Motivation

Currently, when TiDB drops/truncates a table/index, after GC lifetime it will invoke `DeleteRange` on all affected Regions respectively. Though TiKV will call `delete_files_in_range` to quickly free up the space for each Region, the SST files that span over multiple Regions cannot be removed immediately. As a result, the disk space is not able to be recycled in time. Another problem is that the process to delete all Regions respectively is too slow.

This issue has troubled some of our users. It should be fixed as soon as possible.

In our tests, this can finish deleting data of a hundreds-of-GB table in seconds, while the old implementation may cost hours to finish. Also the disk space is released very fast.

# Detailed design

1. Add related protobuf for `UnsafeDestroyRange` to kvrpcpb.
    ```protobuf
    message UnsafeDestroyRangeRequest {
        Context context = 1;
        bytes start_key = 2;
        bytes end_key = 3;
    }
    ```
    The `context` in the request body does not indicate which Region to work. Fields about Region, Peer, etc. are ignored.

    We keep the `context` here, one of the reasons is to make it uniform with other request. Another reason is that, there are other parameters (like `fill_cache`) in `Context`. Though we don't need to use anything in it currently, it's possible that there will be more parameters added to `Context`, which is also possibly needed by `UnsafeDestroyRange`.

2. When TiKV receives `UnsafeDestroyRangeRequest`, it immediately runs `delete_files_in_range` on the whole range on RocksDB, bypassing the Raft layer. Then it will scan and delete all remaining keys in the range.
    
    * Due to that `UnsafeDestroyRange` only helps TiDB's GC, so we put the logic of handling `UnsafeDestroyRange` request to `storage/gc_worker`.
    * `GCWorker` needs a reference to raftstore's underlying RocksDB now. However it's a component of `Storage` that knows nothing about the implementation of the `Engine`. Either, we cannot specialize it for `RaftKv` to get the RocksDB, since it's just a router. The simplest way is to pass the RocksDB's Arc pointer explicitly in `tikv-server.rs`.
    * We regard `UnsafeDestroyRange` as a special type of GCTask, which will be executed in TiKV's GCWorker's thread:
        ```rust
        enum GCTask {
            GC {
                ctx: Context,
                safe_point: u64,
                // ...
            },
            UnsafeDestroyRange {
                ctx: Context,
                start_key: Key,
                end_key: Key,
                // ...
            },
        }
        ```
    * The logic of `UnsafeDestroyRange` is quite similar to `DeleteRange` in apply worker.

    The story of TiKV is ended here. But to show why this thing is useful, let's continue to see what we will do on TiDB:

3. In TiDB, when you execute `TRUNCATE/DROP TABLE/INDEX`, the meta data of the related table will be modified at first, and then the range covered by the truncated/dropped table/index will be recorded in the system table `gc_delete_range` with timestamp when the `TRUNCATE/DROP` query is executed. TiDB GC worker will check this table every time it does GC. In the current implementation, if there are records in `gc_delete_range` with its timestamp smaller than the safe point, TiDB will delete ranges indicated by these records by invoking `DeleteRange`, and also move these records from `gc_delete_range` to `gc_delete_range_done`. So in the new implementation:

    * When doing `DeleteRanges` (the second step of GC), invoke the new `UnsafeDestroyRange` on each TiKV, instead of running the old `DeleteRange` on each Region. Also we need to add an interface to PD so that TiDB can get a list of all TiKVs.
    * Check the record again from `gc_delete_range_done` after 24 hours (24 hours since it was moved to table `gc_delete_range_done`), and invoke `UnsafeDestroyRange` again. Then this record can be removed from table `gc_delete_range_done`.
    
    And why do we need to check it one more time after 24 hours? After deleting the range the first time, if coincidentally PD is trying to move a Region or something, some data may still appear in the range. So check it one more time after a proper time to greatly reduce the possibility.

# Drawbacks

* It breaks the consistency among Raft replicas.
    * But fortunately, in TiDB, when `DeleteRanges` happens, the deleted range will never be accessed again.
* To bypass the Raft layer, the code might violate the layered architecture of TiKV.
* Sending requests to each TiKV server never occurs in TiDB before. This pattern is introduced to TiDB's code the first time.

# Alternatives

We've considered several different ideas. But in order to solve the problem quickly, we prefer an easier approach.

For example, we tried to keep the consistency of Raft replicas. But in a scenario where TiKV is only used by TiDB, deleted range will never be accessed again. So in fact, if we add an interface that is specific for TiDB, we just needn't keep this kind of consistency. It's safe enough.

# Unresolved questions
