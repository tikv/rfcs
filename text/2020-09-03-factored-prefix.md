# Factored Key Prefix

## Summary

TiKV provides a facility where the client can set a key prefix that will be used for all keys in its requests.

## Motivation

TiKV users need to use key prefixes to avoid conflicts.
In keeping with DRY we should not require a program to add the same prefix everywhere.
This will also reduce the payload sent to TiKV.

## Detailed design

### Prefix Manipulation

TiKV already adds a first byte prefix to all keys.
TiKV adds a 'z' first byte prefix to all data keys. Other prefixes are used for metadata.

When the factored prefix is used, TiKV prepends the first byte prefix to the factored prefix sent by the client and prepends that to all keys that are sent.
All keys are returned with the first byte prefix and factored prefix stripped.

A factored prefix of 0x00 is not allowed. This helps avoid accidentally using null as a factored prefix.

### API

The factored prefix applies to all keys in a connection.
So it is set as a GRPC header "TIKV-PREFIX" with the value set to the key prefix.

### Backwards compatibility

Currently applications do not formally claim a prefix and there is no way to know what prefixes they might use.
We can refert to these applications as V1 (version 1) applications. We can refer to applications that use the prefix as V2 (version 2).
We must avoid mixing the usage of V1 and V2 applications.

V1 applications store their data under a 'z' first byte prefix
V2 applications will store their data under a 'y' first byte prefix

If a V1 application sets "TIKV-PREFIX", this is an error.
A V2 application is one which sets "TIKV-PREFIX".
If a V2 application wants to access the entire V2 key space ('y' first byte prefix), it must set the header "TIKV-PREFIX-ALL".
This supports the use case of performing a transaction across any prefixes. The prefix feature does nothing to restrict access: this must be done via an auth feature.

A configuration setting will be added to TiKV: "prefix-only". If this setting is set
* No V1 applications are allowed. It is an error to not set "TIKV-PREFIX".


## Drawbacks

It is not possible to query both V1 and V2 applications: in other words to query the 'z' first byte prefix and the 'y' first byte prefix.

This proposal was created to aid the existing key spaces proposals. Without implementing key spaces this feature may not get used by users.


## Alternatives

Have the TiKV client set the prefix.
This proposal is equivalent to the client itself prepending the key prefix plus having a setting for V2 clients to store under a new 'y' first byte prefix.
To satisfy the DRY principle, we could make prefix prepending a standard feature of all TiKV client libraries.

However, we won't get the benefit of
1) reduced payload to TiKV
2) the server helping to ensure key prefixes are used
3) improved performance in the client by avoiding prepending

For the last point, it may seeem that this simply moves work from the TiKV client to the TiKV server.
However, TiKV already performs prefix manipulation for all keys so we expect the additional work in TiKV to be less noticeable.

## Unresolved questions

Please ask your questions.
