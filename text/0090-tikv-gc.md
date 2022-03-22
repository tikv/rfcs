# RFC: TiKV RawKV MVCC GC


## Summary
Move TiKV MVCC GC worker from TiDB into a group of independent GC worker node role and implement a new GC process in TiKV for RawKV.

## Motivation
GC worker is an important component for TiKV that deletes outdated MVCC data so as to not explode the storage. But currently, the GC worker is implemented in TiDB, which makes TiKV not usable without TiDB.And current GC process is just for transaction of TiDB,it's not usable for RawKV.

## Background
According to the documentation for the current GC worker in a TiDB cluster, the GC process is as follows:

In TiDB GC worker leader:
1. Regularly calculates a new timestamp called "safe point", and push the safe point to PD.
2. Get the minimal Service safe point  among all services from the response of step 2, which is GC safe point .
3. Txn GC process: resolve locks and record delete ranges information.

In PD leader:
1. Receive update safe point requests from TiDB or other tools (e.g. CDC, BR).
2. Calculate the minimal timestamp = min(all service safe point, now - gc_life_time).

In every TiKV nodes：
1. Get GC safe point from PD regularly.
2. Deletion will be triggered in CompactionFilter and GcTask thread;
   
## New GC worker architecture
In a TiKV cluster without TiDB nodes , there are a few different points as follows:
1. We need to move GC worker into another node role.
2. For [API V2](https://github.com/tikv/rfcs/blob/master/text/0069-api-v2.md) .It need gc the earlier version in default cf.But Txn GC worker process will be triggered by WriteCompactionFilter of write cf.
3. RawKV encoded code of RawValue is different with Txn in TiDB.

So we designed a new GC architecture and process for TiKV cluster.

## Detailed design
For support TiKV cluster deploy without TiDB nodes.
1. Add a new node role instead of GC worker in TiDB nodes.
   The code of new GC worker,It will be added into [tikv/migration](https://github.com/tikv/migration)
2. And for API V2, we need add new CompactionFilter which is named RawGCcompactionFilter, and add a new GCTask type implementation.
3. GC conditions in RawGCcompactionFilter is:  (ts < GCSafePoint) && ( ttl-expired || deleted-mark || not the newest version ).  
   1. If the newest version is earlier than GC safe point and it's delete marked or expired ttl,those keys and earlier versions of the same userkey will be sent to a gc scheduler thread to gc asynchronous.

## Reference
https://docs.google.com/document/d/1jA3lK9QbYlwsvn67wGsSuusD1Dzx7ANq_vya384RBIg/edit#heading=h.rr3hcmc7ejb8  
https://docs.pingcap.com/tidb/stable/garbage-collection-overview
