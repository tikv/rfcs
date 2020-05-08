# Title

## Summary

This RFC proposes to forbid users using RawKV API on the TiKV cluster by default,
and provide a way for users to enable it.

## Motivation

Some painful incidents happened before and still happening,
because of the mixed-using of RawKV interface and TransactionKV interface,
some key-values were overwritten by RawKV,
which originally written by TransactionKV, then led to data corruption.

We should add more warnings about this in our documents,
but it's not enough, documents could easily be ignored.

Disable RawKV by default could prevent people from misuse without noticing,
when users need to step into the danger zone,
they need to explicitly open the guarded switch by command.
And they could shut down the switch anytime to gain more safety.

## Detailed design

Add a `kv-mode` flag to the cluster, this flag should be persisted,
each member of the TiKV cluster should see the same value at the same time,
so it could be:

- Store in a special key in the underlying engine and use a new command
  to manipulate it
- Use the dynamic configuration feature to achieve our goal.

Since the dynamic configuration is already supported, it should be good for this.

`kv-mode` should be an enum:

- `ModeTxnKV` indicates that the RawKV commands could not be called.
  When they are called they will return errors.
  This is the default value.
- `ModeRawKV` indicates that the TxnKV commands could not be called.

Provide a config entry `mode` under `[server]` in the config file.
It might be necessary when we trying to rolling-update a cluster which already
running mixed API.

## Drawbacks

The VerKV commands should not be affected by the `kv-mode` flag,
because they use different keyspace and could be mixed-using with
either RawKV or TxnKV commands.
So, the semantic meaning of the `kv-mode` flag will be a little confusing.

## Alternatives

Modified the RawKV implement, let them manipulate keys in
different keyspace as VerKV did.

- This will impact data compatibility, especially on cluster upgrading.
- Use RawKV to manipulate TxnKV data sometimes could be useful, eg, data fixing.

## Unresolved questions

None.
