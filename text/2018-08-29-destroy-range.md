# Summary

Support RPC `DestroyRange`. This call is on the whole TiKV rather than a certain Region. When it is invoked, TiKV will use `delete_files_in_range` to quickly free a large amount of space, and then scan and delete all remaining keys in the range. Raft layer will be bypassed, and the range can cover across multiple Regions. The invoker should promise that after invoking `DestroyRange`, the range will never be accessed again. That is to say, the range is permanently scrapped.

This interface is only designed for TiDB. It's used to clean up data after dropping/truncating a huge table/index.

# Motivation

Currently, when TiDB drops/truncates a table/index, after GC lifetime it will invoke `DeleteRange` on all affected Regions respectively. Though TiKV will call `delete_files_in_range` to quickly free up the space for each Region, the SST files that span over multiple Regions cannot be removed immediately. As a result, the disk space is not able to be recycled in time. Another problem is that the process to delete all Regions respectively is too slow.

This issue has troubled some of our users. It should be fixed as soon as possible.

# Detailed design

1. Add related protobuf for `DestroyRange` to kvrpcpb.
    ```protobuf
    message DestroyRangeRequest {
        bytes start_key = 1;
        bytes end_key = 2;
    }
    ```
    It does not contain a context to indicate which Region to work on.

2. When TiKV receives `DestroyRangeRequest`, it immediately runs `delete_files_in_range` on the whole range on RocksDB, bypassing the Raft layer. Then it will scan and delete all remaining keys in the range.
    > TODO: Describe more detailed implementation here

    The story of TiKV is ended here. But to show why this thing is useful, let's continue to see what we will do on TiDB:

3. In TiDB, when you execute `TRUNCATE/DROP TABLE/INDEX`, the meta data of the related table will be modified at first, and then the range covered by the truncated/dropped table/index will be recorded in the system table `gc_delete_range` with timestamp when the `TRUNCATE/DROP` query is executed. TiDB GC worker will check this table every time it does GC. In the current implementation, if there are records in `gc_delete_range` with its timestamp smaller than the safe point, TiDB will delete ranges indicated by these records by invoking `DeleteRange`, and also move these records from `gc_delete_range` to `gc_delete_range_done`. So in the new implementation:

    * When doing `DeleteRanges` (the second step of GC), invoke the new `DestroyRange` on each TiKV, instead of running the old `DeleteRange` on each Region.
    * Check the record again from `gc_delete_range_done` after 24 hours (24 hours since it was moved to table `gc_delete_range_done`), and invoke `DestroyRange` again. Then this record can be removed from table `gc_delete_range_done`.
    
    And why do we need to check it one more time after 24 hours? After deleting the range the first time, if coincidentally PD is trying to move a Region or something, some data may still appear in the range. So check it one more time after a proper time to greatly reduce the possibility.

# Drawbacks

* It breaks the consistency among Raft replicas.
    * But fortunately, in TiDB, when `DeleteRanges` happens, the deleted range will never be accessed again.
* To bypass the Raft layer, the code might violate the layered architecture of TiKV.

# Alternatives

We've considered several different ideas. But in order to solve the problem quickly, we prefer an easier approach.

For example, we tried to keep the consistency of Raft replicas. But in a scenario where TiKV is only used by TiDB, deleted range will never be accessed again. So in fact, if we add an interface that is specific for TiDB, we just needn't keep this kind of consistency. It's safe enough.

# Unresolved questions
