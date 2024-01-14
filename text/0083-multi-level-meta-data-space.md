# Multi-level Meta Data Space

## Summary

PD can handle multiple users' meta data, each user only see its own meta data.

## Motivation

PD is the mata data center of TiKV Cluster, the meta data contains TiKV instances' addr, user data range(region)
information, the number of replicas of data, etc.

But PD currently only can handle the key space [min-key, max-key], it means that PD only can handle one user's 
meta data, it is not possible when multiple users want share the same PD cluster as their meta system. There are
two different scenarios that require PD can handle multiple user' meta data:

1. Multiple TiKV Cluster share the same PD cluster. Because the minimal demplyment of a TiKV Cluster is 3 TiKV 3 PD,
but it is not cost-effect if every small cluster has 3 dedicated meta data node.
2. There are Multiple tenant in the same TiKV Cluster, each tenant has it own meta data, each tenant's key range can
contains any key in the range of [min-key, max-key].

## Detailed design

Change the meta data from
```
{meta data}
```
to
```
user1 {meta data}
user2 {meta data}
user3 {meta data}
...
```

### Compatibility

When upgrade from old version, all legacy meta data belongs to the default meta data space.
```
{meta data}
```
```
default {meta data}
user1 {meta data}
user2 {meta data}
...
```

## Alternatives

In the multi-tenant scenario, tenant can add a {tenant-id} prefix for each data key, but tenant-id
is a meta data esstionally, each data key has a tenant-id prefix may cost more disk space & memory
in TiKV.
