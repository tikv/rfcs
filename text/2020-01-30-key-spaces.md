# Title

Key Spaces

## Summary

When multiple applications store data in TiKV, we need a clear pattern for avoiding conflicting use of the key space in TiKV.
TiKV is a monolithic KV store, so each application must choose a key prefix.
Key Spaces formalizes this process by registering a key space for an application in TiKV itself.

When combined with authentication (not part of this proposal), this can secure an application's key space.

Additionally, key spaces make it possible for TiKV to avoid conflicts during a data restore by changing the key prefix assigned to a key space.

## Motivation

The use case for this is running multiple applications against TiKV (not just a single TiDB).
Note that we may be motivated to run multiple instances of a single application such as TiDB.

* Avoiding even accidental conflicting use of key space in TiKV is currently ad-hoc and unreliable
* Securing data in TiKV is difficult. A necessary first step is clearly defining the boundaries of data.

Once key spaces are in place, all applications and TiKV itself should understand what data belongs to whom and how to avoid key space conflict.

## Detailed design

#### Terminology

This proposal allows the total TiKV key space to be divided into individudal key spaces for applications.
To refer to the total TiKV key space we will use the terminology "total TiKV key space".

#### Scope

This RFC will not propose "sub spaces", which would be a key space within a key space.
I think this is a good idea, but I also think it can simply be proposed independently, although it may be better to discuss after the key spaces RFC is accepted so that it can leverage some of the key space implementation.

This RFC will not discuss authentication.
Authentication is necessary to provide any meaningful security for key spaces, but I can start a separate RFC for this.

#### Client view

When a client first connects to the TiKV system, it will register itself to get a key space.
This happens in the metadata component (PD).
All requests to the storage component (TiKV) will include the key space.
The TiKV storage component will transparently convert the key space into a key prefix that TiKV uses for its operations.

The client can register for additional key spaces.

#### Metadata operations

POST /keyspaces

requires:

* application name

optional:

* key space name
* description
* prefix

This API will return the key space object, which includes an ID. Key spaces have an ID to allow their name to be changed.
Clients are recommended to provide a key space name, but the API can also auto-generate one.

The usage of an application name as an identity may end up being a de-normalization of data in a future iteration when identity is added for authentication and there are identity ids.
I suggest that we can add an identity id as an alternative field to an application name. Similarly, the output key space object can include the identity id.

Addtional GET APIs will also return keyspace objects

* Lookup by ID, key space name
* List all key spaces

POST /keyspaces/{id}

requires id and description. Only the description field can be updated.

DELETE /keyspaces/{id}

requires id. A key space may only be deleted after all its used keys are deleted.
Before the delete operation is completed, it requires PD to first notify all TiKV that would receive requests for the key space of the deletion.

#### TiKV storage operations

Requests to TiKV will have a new field "key space", which is filled with the key space id.
TiKV looks up the key space id in PD to translate it to a key prefix.
TiKV will then prepend the key prefix to all key usage in the request.
It will strip the prefix from returned keys.
Note that this lookup with PD will be cached.
The cache can only expire if the key space is deleted because the key space id and prefix do not change.

#### Key Prefix specification or compression

Normally, a client registers for a keyspace with a name, but the actual key prefix is transparently assigned, known only to TiKV.
This allows TiKV to change the prefix when doing a restore and also allows it to perform key prefix compression.
Note that some storage engines may already be able to perform prefix compression.
For backwards compatibility purposes, a client is also allowed to be registered with an explicit key prefix.

#### The default key space

A TiKV client may ignore key spaces. In a fresh TiKV deployment with this feature in place, that client will be using the default key space.
A configuration option `disable-default-key-space` when set to `true` forces clients to specify a key space rather than using the default key space. 

#### Backwards compatibility

For an existing deployment we will need an upgrade process.
We can have a configuration flag "allow-unmanaged-key-space".
For a fresh deployment this will be `false`, but if there is an existing deployment it should be `true`.
When true, clients that do not specify a key space will continue using the key space the same way they do now.
When false, clients that do not specify a key space will use the default key space.

Clients can be transitioned over to the keyspace system by registering them for a key space and specifying that the current key prefix that they use must be used. 
At that point they can begin to use the registered key space and stop prepending a prefix.

Once all clients are transitioned, "allow-unmanaged-key-space" can be set to `false`.

#### Backup and Restore

A full backup should backup all the keyspace metadata.
A backup of a single key space should backup the information about the associated keyspace.
A restore that has associated key space metadata must also ensure that after the restore there are key space metadata entries in PD.

When restoring, TiKV may choose to alter the key prefix.
Suggested best behavior for restore is to be conservative about conflicts and to let the user handle conflicts:

* check to see if the key prefix for the key space is already in use. If so, this is an error
* the user may instead provide a flag to tell TiKV to overwrite the keyspace or alter the key prefix

#### Ownership and multiple applicatins

The intent of key spaces is to declare an owner of a key space.
There can only be one owner.
This does not preclude another application from making use of a key space.
It would be possible for this proposal to attempt to formalize this as well by documenting read-only access, etc.
However, I believe that because currently we have no way to secure access it is better to tackle this problem when we implement authentication.
More specifically, there are alternative designs for this that make more sense when authentication is present, and one that I currently favor is access delegation.

## Drawbacks

The upgrade path is a little complicated and might be confusing.
Teaching users to use this new feature and attempting to integrate it into client libraries will take effort.
key space overwriting is still possible by accidentally copy & pasting a key space name to a new application instead of creating a new one.

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
