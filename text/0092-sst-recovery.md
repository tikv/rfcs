# Online SST Recovery

- RFC PR: [https://github.com/tikv/rfcs/pull/92](https://github.com/tikv/rfcs/pull/92)
- Tracking Issue: [https://github.com/tikv/tikv/issues/10578](https://github.com/tikv/tikv/issues/10578)

## Summary

When part of the SST files is corrupted or inaccessible and the errors are background, TiKV should not panic immediately. Damaged SSTs should be automatically deleted.

## Motivation

When SST files are corrupted or inaccessible, TiKV would panic and cannot start normally which would block the user’s business. In such situations, we need to intervene to manually help users recover. 

Often the failure like this is caused by IO error or OS error. If the damaged store cannot provide services, the above leaders cannot handle read and write requests, which block the user’s business.

## Detailed design

RocksDB provides a hook for background error, and TiKV's current approach is to panic directly here. In this design, We can use the hook to run SST recovery worker when SST corrupted or inaccessible errors. 

When the data belonging to the data range is damaged, it will be reported to PD through heartbeat, and PD will add `remove-peer` operator to remove this damaged peer. When the damaged peer still exists in the current store, the corruption SST files remain, and the KV storage engine can still put new content normally, but it will return error when reading corrupt data range.

If after max recovery time window, the peer where the corrupted data range located has not been removed from the current store, TiKV will panic.
