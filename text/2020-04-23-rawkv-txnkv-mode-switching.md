# Title

## Summary

This RFC proposes to forbid users using RawKV API on the TiKV cluster by default and provide a new API command to let users enable it.

## Motivation

Some painful incidents happened before and still happening, because of the mixed-using of RawKV API and TransactionKV API, some key-values were overwritten by RawKV API, which originally written by TransactionKV API, then led to data corruption.

We should add more warnings about this in our documents, but it's not enough, documents could easily be ignored.

Providing a switching API could prevent people from doing this wrong thing without noticing, when users need to step into the danger zone, they need to explicitly open the guarded switch. And they could shut down the switch anytime to gain more safety.

## Detailed design

This `mode` flag should be persisted, each member of the TiKV cluster should see the same value in the same time, so it should be write/read by RawKV commands. Or we just use the dynamic configuration feature to achieve our goal.

`mode` should be an enum:
- `ModeRawKV`, indicate that the TxnKV commands could not be called. When they are called they will return errors.
- `ModeTxnKV`, indicate that the RawKV commands could not be called.
- `ModeTxnKVRawKV`, works as now.
The default value of `mode` should be `TxnKV`

Provide a config entry `mode` under `[server]` in the config file.
It's necessary when we trying to rolling-update a cluster which already running mixed API.

Add a pair of command `Get/SetMode(mode)` to the API.

## Drawbacks

The VerKV commands should not be affected by the `mode` flag, because they use different keyspace and could be mixed-using with either RawKV or TxnKV commands.
So, the semantic meaning of the `mode` flag will be a little confusing.

## Alternatives

Modified the RawKV implement, let them manipulate keys in different keyspace as VerKV did.
- This will impact data compatibility, especially on cluster upgrading.
- Use RawKV to manipulate TxnKV data sometimes could be useful, eg, data fixing.

## Unresolved questions

None.
