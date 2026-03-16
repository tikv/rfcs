# Coprocessor Write

## Summary

This RFC intends to extend the ability of coprocessor to support writing. The user can send a _single_ coprocessor request to do a TiDB query (a.k.a. a DAG request) first and then perform a write request (like prewrite or pessimistic lock) with the result of the previous TiDB query.

## Motivation

Currently, all write requests are initiated from a TiKV client. To update some data, the user typically reads from TiKV first and prepares the data to write. And then, it sends a write request to TiKV. The whole process includes at least two RPCs between the client and TiKV. There is usually extra cost of encoding and decoding.

However, in multiple cases, we can move some of the logic from the client to TiKV to reduce one RPC and save encoding/decoding cost.

### Simple updates

If the result set of a DAG request is the same as the data to write, we can let coprocessor invoke prewrite.

Suppose we have a typical table with a primary key and secondary indexes:

| Column | Type | Key             |
| ------ | ---- | --------------- |
| id     | INT  | primary key     |
| v      | INT  |                 |
| idx    | INT  | secondary index |

If we update the non-index column `v` according to the primary key `id`, the rows we query are right the rows we are going to write.

For example, if we do `UPDATE t SET v = v + 1 WHERE id = 1`, we can query TiKV with `SELECT id, v + 1, idx FROM t WHERE id = 1`. Then, the result row is identical to the row we want to write. It will be easy for coprocessor to also do a prewrite and return the prewrite result to the client.

### Pessimistic lock rows with conditions

In pessimistic transactions, keys to be written need to be locked before the 2PC process. Only mutations with known row handle can be locked with a single RPC. In other cases, we need two RPCs: the first one to get the rows and the second one to lock the rows.

Using the same table in the previous section, when we want to do `UPDATE t SET idx = idx + 1 WHERE v BETWEEN 10 AND 20`, we do a TiDB query with `SELECT * FROM t WHERE v BETWEEN 10 AND 20` first, and then we lock the rows with `AcquirePessimisticLock` RPCs.

We can find the result set contains sufficient information to construct the row keys we want to lock. It would be better if we can use only one RPC to do the query and lock these keys. Coprocessor can take the result set of the query as the input of `AcquirePessimisticLock`. This can save one RPC.

## Detailed design

TiKV coprocessor has three types of endpoint requests now: DAG, ANALYZE and CHECKSUM. This RFC proposes to add two more types: DAG_PREWRITE and DAG_PESSIMISTIC_LOCK.

### DAG_PREWRITE

DAG_PREWRITE is a combination of a DAG request and a prewrite request. The whole transaction should span only one region.

It first executes the DAG request to generate the rows to write. The rows are encoded to key-value pairs and prewritten. All the options of prewrite (except the start TS) are chosen by TiKV and will be returned in the coprocessor response.

It is suitable for updating non-index fields of a table in an auto-commit transaction.

#### Type ID

DAG_PREWRITE sets the `tp` field to `110` in the coprocessor request. (106~109 are reserved for other read only requests.)

#### Request format

The `data` field in the request is a `DagRequest` message. But the `DagRequest` should follow these restrictions:

* The `DagRequest` must be based on a `TableScan`.
* The `DagRequest` must not contain aggregation executors.
* The `DagRequest` must not change primary key columns.
* The `DagRequest` should contain a `Projection` executor which retains the schema of the table.

#### Response format

The `data` field in the response contains a `PrewriteRequest` and a `PrewriteResponse`:

```
|len1|PrewriteReq|len2|PrewriteResp|
|----|-----------|----|------------|

len1:         length of PrewriteReq, 64-bit unsigned integer
PrewriteReq:  encoded PrewriteRequest protobuf message
len2:         length of PrewriteResp, 64-bit unsigned integer
PrewriteResp: encoded PrewriteResponse protobuf message
```

`PrewriteReq` includes necessary information for the client to finish the 2PC:

* List of keys
* Primary key
* Whether 1PC is used

`PrewriteResp` is the response of the prewrite operation.

Errors related to the execution of DAG request are set in the error fields in the coprocessor response. Prewrite related errors are set in `PrewriteResp`.

#### Projection executor

Now TiKV does not support the projection executor. We should have a basic implementation of it to support DAG_PREWRITE. 

#### Future optimizations

The initial implementation tends to be simple and general. So we reuse the current DAG execution and prewrite implementation. The problem is that we seek the key in the DAG execution and seek a second time in prewrite.

If the update ranges are all point ranges, it is possible to lock these keys in the latch first and do the constraint checks of prewrite during the `TableScan`.

This optimization needs some more work. We can create a `PointPrewrite` executor which is a specialized `TableScan` as the bottom executor of DAG. It yields results just like `TableScan` but also does constraint checks of prewrite. After the execution, we are safe to send the mutations to the underlying raft store without checking write conflicts again.

### DAG_PESSIMISTIC_LOCK

DAG_PESSIMISTIC_LOCK is a combination of a DAG request and an AcquirePessimisticLock command. It is suitable for `UPDATE` or `SELECT FOR UPDATE` with simple conditions.

It first executes the DAG request to generate the rows to lock. The rows are encoded to record keys.

Most of the options of AcquirePessimisticLock, should be provided by the client. The result of the DAG request extends the keys to lock. If `primary_lock` is not provided, the first key is selected as the primary key. So it also means that the first statement of a pessimistic transaction cannot use DAG_PESSIMISTIC_LOCK if it involves multiple regions.

#### Type ID

DAG_PESSIMISTIC_LOCK sets the `tp` field to 111 in the coprocessor request.

#### Request format

The `data` field in the response contains a `DagRequest` and a `PessimisticLockRequest`:

```
|len1|DagRequest|len2|PessimisticLockReq|
|----|----------|----|------------------|

len1:               length of DagRequest, 64-bit unsigned integer
DagRequest:         encoded DagRequest protobuf message
len2:               length of PessimisticLockReq, 64-bit unsigned integer
PessimisticLockReq: encoded PessimisticLockRequest protobuf message
```

Currently we only support the simplest case. The `DagRequest` should not contain aggregation executors or projection executors.

#### Response format

The `data` field in the response contains a `SelectResponse` and a `PessimisticLockResponse`:

```
|len1|SelectResponse|len2|PessimisticLockResp|
|----|--------------|----|-------------------|

len1: length of SelectResponse, 64-bit unsigned integer
SelectResponse: encoded SelectResponse protobuf message
len2: length of PessimisticLockResp, 64-bit unsigned integer
PessimisticLockResp: encoded PessimisticLockResponse protobuf message
```

`SelectResponse` is the response of the DAG request. `PessimisticLockResp` is the response of AcquirePessimisticLock.

Errors related to the execution of DAG request are set in the error fields in the coprocessor response. Pessimistic lock related errors are set in `PessimisticLockResp`.

## Drawbacks

To be discussed.

## Alternatives

The alternative way is to integrate expression executions to the prewrite or pessimistic lock command. But it is less optimal because TiKV is essentially a KV store thus non-KV features should be always done in the coprocessor.

## Unresolved questions

To be discussed.
