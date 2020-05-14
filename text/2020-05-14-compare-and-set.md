# Compare-And-Set

## Summary

It allows to conditionally update a key, the condition being that the key has a certain value(maybe not exist).

## Motivation

It allows to implement read-modify-write scenarios.

## Design

- We can run the Compare-And-Set operation in the Scheduler which use Latch to control write concurrency.
- If there is conflicts, return fail immediately or wait in the queue.


### RPC Interface

```
message RawCheckAndSetRequest {
    Context context = 1;
    bytes key = 2;
    bytes old_value = 3;
    bytes new_value = 4;
    string cf = 5;
}

message RawCheckAndSetResponse {
    errorpb.Error region_error = 1;
    string error = 2;
}

enum CmdType {
    Invalid = 0;
    Get = 1;
    Put = 3;
    Delete = 4;
    Snap = 5;
    Prewrite = 6;
    DeleteRange = 7;
    IngestSST = 8;
    ReadIndex = 9;
    CheckAndSet = 10;
}
```

### RawKVScheduler

```
pub struct RawLatches {
    slots: Vec<RwLock<Latch>>,
    size: usize,
}
```

- The key of a CheckAndSet command are hashed, then try to acquire the writer lock if the command ID is the first one of elements in the queue.
- The key of other RawKV write commands are hashed and try to acquire the reader lock.


### Modify::CheckAndSet

```
pub enum Modify {
    Delete(CfName, Key),
    Put(CfName, Key, Value),
    DeleteRange(CfName, Key, Key, bool),
    CheckAndSet(CfName, Key, OldValue, NewValue),
}
```

### RocksDB
- Merge Operator, the Atomic Read-Modify-Write operation in RocksDB.


## Alternatives

- Use Txn Api, get and set

## Unresolved questions
