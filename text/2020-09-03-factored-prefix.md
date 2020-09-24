# Factored Key Prefix

## Summary

TiKV provides a facility where the client can set a key prefix that will be used for all keys in its requests.

## Motivation

TiKV users need to use key prefixes to avoid conflicts.
In keeping with DRY (Don't Repeat Yourself) we should not require a program to add the same prefix everywhere.
This proposal aids the key spaces proposal.

## Detailed design

### Prefix Manipulation

TiKV already adds a first byte prefix to all keys.
TiKV adds a first byte prefix (currently 'z') to all data keys. Other prefixes are used for metadata.

When the factored prefix is used, TiKV prepends the first byte prefix to the factored prefix sent by the client and prepends that to all keys that are sent.
All keys are returned with the first byte prefix and factored prefix stripped.

A factored prefix of 0x00 is not allowed. This helps avoid accidentally using null as a factored prefix.

### API

The factored prefix applies to all keys in a connection.
So it is set as a GRPC header "TIKV-PREFIX" with the value set to the key prefix.

### Backwards compatibility

Currently applications do not formally claim a prefix and there is no way to know what prefixes they might use.
We can refer to these applications as V1 (version 1) applications. We can refer to applications that use the prefix as V2 (version 2).
We must avoid mixing the usage of V1 and V2 applications.

A V2 application is one which sets "TIKV-PREFIX".

To avoid conflicts with a V1 application in the V2 system, the V1 application should continue to use only the prefixes it is currently using. For TiDB, this is both 'm' and 't'. Future versions of TiDB that add new prefixes allow them to be configurable to avoid collisions.

A V1 application can continue to behave as a V1 and make global requests without setting "TIKV-PREFIX".
If the V1 application uses just one prefix it is recommended to immediately change to V2 behavior of settinging "TIKV-PREFIX".


### Prefix only

A configuration setting will be added to TiKV: "prefix-only".
* No V1 application access is allowed
    * It is an error to not set "TIKV-PREFIX".

This is useful for a multi-tenant situation where there should be no access across tenants.


## Drawbacks

This proposal was created to aid the existing key spaces proposals. Without implementing key spaces this feature may not provide enough value on its own.


## Alternatives

Have the TiKV client add the prefix to all its request keys (as it does today).
This proposal is equivalent to the client itself prepending the key prefix.
To satisfy the DRY (Don't Repeat Yourself) principle, we could make prefix prepending a standard feature of all TiKV client libraries.

However, we won't get the benefit of
1) the server helping to ensure key prefixes are properly used
2) improved performance in the client by avoiding prepending

For the last point, it may seeem that this simply moves work from the TiKV client to the TiKV server.
However, TiKV already performs prefix manipulation for all keys so we expect the additional work in TiKV to be less noticeable.

## Unresolved questions

Please ask your questions.
