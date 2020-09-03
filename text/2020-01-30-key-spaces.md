# Title

Key Spaces

## Summary

When multiple applications store data in TiKV, we need a clear pattern for avoiding conflicting use of the key space in TiKV.
TiKV is a monolithic KV store, so each application must choose a key prefix.
Key Spaces formalizes this process by registering a key space for an application in TiKV itself.

When combined with authentication (not part of this proposal), this can secure an application's data.

Additionally, key spaces make it possible for TiKV to avoid conflicts when an application is restored into a different cluster by changing the key prefix assigned to a key space.

## Motivation

The use case for this is running multiple applications against TiKV (not just a single TiDB application).
This gives the user 2 potential benefits:
* Reduce maitenance burden by operating a single cluster.
* Reduce cost and otherwise make TiDB more feasible for smaller data sets

The second point is critical for TiDB to be able to serve a broader set of users.

The problem with running multiple applications is conflicting, unsecured key spaces.
* Avoiding accidental conflicting use of a key space in TiKV is currently ad-hoc and unreliable
* Securing data in TiKV against privilege escalation. If one TiKV client is hacked, effectively all are hacked.

Once key spaces are in place, all applications and TiKV itself should understand what data belongs to whom and how to avoid key space conflict.
An auth implementation can then secure the key spaces to provide security against privilege escalation.

## Detailed design

#### Scope

This RFC will not propose "sub spaces", which is a key space within a key space.
This can be proposed independently, although it may be better to discuss after the key spaces RFC is accepted so that it can leverage some of the key spaces implementation.

This RFC will not discuss authentication.
Authentication is necessary to provide any meaningful security for key spaces.

This RFC will not discuss resource isolation.
Key spaces will be an important tool for resource isolation to leverage.

#### Client view

When a client first connects to the TiKV system, it will register itself to get a key space prefix.
The client can register for additional key spaces, but normally one application will have one key space.

All key spaces operations take place in the metadata component (PD) which stores a registry of key spaces.

The TiKV component does not know about key spaces. All requests to the storage component (TiKV) will include the key space prefix.
Please see [this separate proposal that helps with sending key prefixes to TiKV](https://github.com/tikv/rfcs/pull/56).


#### Metadata operations

POST /keyspaces

requires:

* key space name
* application name

optional:

* description

system generated:

* prefix

This API will return the key space object. The key space name is the id of the keyspace and cannot be changed.
The usage of an application name as an identity may end up being a de-normalization of data in a future iteration when true identity with authentication is added.
The prefix that is generated is a binary sequence. This will start with a single byte sequence: this optimizes for the fact that few TiKV installations will have more than 255 applications.
The sequence starts with 0x01 and increment by 1. The 0x00 prefix will remain reserved.
A byte sequence containing 0xFF for all bytes is not a valid prefix because this is the end of the range. When a sequence contains just 0xFF, a byte of 0x00 is appended and the byte sequence is now one byte longer. This pattern allows us to start with just a 1 byte sequence but allows expansion to any length.


POST /keyspaces/{id}

requires id and description. Only the description field can be updated.

Addtional GET APIs will also return keyspace objects

* Lookup by ID, key space name
* List all key spaces

DELETE /keyspaces/{id}

requires id. Deletion is background task. This DELETE API sets the time of the API call in a field `deleted_at`.
When deletion is completed, it will set a field `delete_completed_at` and the prefix may be re-used in a new key space.

Note that in a token-based auth system deletion will need to wait until client tokens expire.



#### Key Prefix specification or compression

PD generates key prefixes. PD will generate a 4 byte prefix. The first byte will be set to 0xFF
A client registers for a keyspace with a name, but the actual key prefix is different.
This allows TiKV to change the prefix when doing a restore and also allows for a short key prefix to be generated.

#### Backwards compatibility

Please see [this separate proposal that helps with sending key prefixes to TiKV](https://github.com/tikv/rfcs/pull/56).
This includes a backwards compatibility plan.
If that RFC is not adopted we would likely do something similar in this RFC (but particular to key spaces rather than prefixes).

Otherwise this RFC provides no help with backwards compatibility. Applications not using the key spaces feature will stay the same.
The auth proposal will provide a mechanism to secure access for an application that did not start by using the key spaces feature.


#### Backup and Restore

A full backup should backup all the keyspace metadata.
A backup of a single key space should backup the information about the associated keyspace.
A restore that has associated key space metadata must also ensure that after the restore there are key space metadata entries in PD.

When restoring, TiKV may choose to alter the key prefix to avoid conflicts. A conflict could happen when moving an application from one cluster to another.
Suggested best behavior for restore is to be conservative about conflicts and to let the user handle conflicts:

* check to see if the key prefix for the key space is already in use. If so, this is an error
* the user may instead provide a flag to tell TiKV to overwrite the keyspace or alter the key prefix

#### Ownership and multiple applicatins

The intent of key spaces is to declare an owner of a key space.
There can only be one owner.
This does not preclude cooperation among applications, for example having read access to another key space.

It would be possible for this proposal to attempt to formalize this as well by documenting read-only access, etc.
However, I believe that because currently we have no way to secure access it is better to tackle this problem when we implement authentication.
More specifically, there are alternative designs for this that make more sense when authentication is present, and one that I currently favor is access delegation.

## Drawbacks

The upgrade path is a little complicated and might be confusing.
Teaching users to use this new feature and attempting to integrate it into client libraries will take effort.
key space overwriting is still possible until auth is implemented.

## Alternatives

#### FoundationDB directories

The database I know of that is operationally similar to TiKV is FoundationDB. FoundationDB has had this feature for a long time.
FoundationDB has a tuple layer which is a convenience for creating key prefixes. On top of that, they have subspaces, which is specifically designed for prefixes for key space managment.
In addition, they have a directory layer.
The directory layer abstracts away actual key prefixes, usess a compressed key prefix, and has some hierarcrhy abilities for organizing data.

The Directory layer has been problematic for some users precisely because it is a "layer" rather than implemented directly in FoundationDB.
Fundamentally use of this layer requires an extra database read or the use of a cache that you can't guarantee is correct unless you control the usage of your directories and are aware of backup and restore, etc.

So this proposal is similar to FoundationDB directories, but

* does not attempt to add hierachical organization
* is implemented directly in the TiKV system to avoid network overhead

I believe that if the hierarchy feature is desired it can be added on to this key space implementation.
In a database, it is very important to avoid abstractions that make performance worse, so I believe this is a feature that should not be implemented as a (network-isolated) layer.

FoundationDB subspaces are little more than a convenience in client libraries for using key prefixes.
Etcd namespaces seem equivalent.
We may also want to add such a feature to TiKV clients independently on this RFC.

#### Running lots of TiKVs

Instead of key space partitioning, one could run one TiKV cluster per applcation.
This can work well, and may still be a preferred approach as long as each application's usage is large enough to justify a TiKV deployment.
Unfortunately TiKV is not well-tested for scaling down to smaller resource deployments right now.
Additionally, once deployed on a smaller sized node, vertical scaling is complicated.
It is also possible to deploy many small TiKV on one large node, but similar issues are present.
I can elaborate more on this situation if there is interest, but otherwise will spend my time on this proposal.

Operationally, users may prefer running 1 large TiKV cluster over running many smaller clusters.

But I don't think these 2 approaches are exclusive: we need to work to support both the ability to run multiple applcations in one TiKV deployment and the ability to manage multiple TiKV cluster deployments.

## Unresolved questions
