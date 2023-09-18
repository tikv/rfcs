# Kubernetes Operator For TiKV

## Summary

This doc proposes to split out TiKV Operator from [TiDB
Operator](https://github.com/pingcap/tidb-operator) so that we could provide a
TiKV centric operator for Kubernetes with less external dependencies from TiDB.

## Motivation

It's challenging to manage stateful applications in production. To properly
scale and upgrade TiKV stores, users need to know application's operational
domain knowledge. With the help of [Kubernetes
Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/), we
can program TiKV specific knowledge into an operator to automate many tasks in
Kubernetes.

TiDB Operator focuses on TiDB scenarios and TiDB server is not an optional
component. It's complex and users can not use it to deploy and manage TiKV
clusters only in Kubernetes, e.g. use TiKV RawKV for Key-Value storage.

## Detailed design

Create a TiKV Operator project which focuses on TiKV deploy, upgrade, and
automation in Kubernetes.

A [Custom
Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
can be implemented to declare the desired state of a TiKV cluster, e.g.

```yaml
apiVersion: tikv.org/v1alpha1
kind: TikvCluster
metadata:
  name: example
spec:
  version: v4.0.0
  pd:
    baseImage: pingcap/pd
    replicas: 3
    config: {}
  tikv:
    baseImage: pingcap/tikv
    replicas: 3
    config: {}
```

In the initial stage, we will follow the design of TiKV spec in the TiDB Operator.
In the future, TiDB Operator can adopt TiKV Operator to manage TiKV in a TiDB
cluster.

## Drawbacks

- We must maintain a project for TiKV Operator and keep it up to date as TiKV
  evolves

## Alternatives

There are some other solutions to deploy `tikv-server` without TiDB in Kubernetes:

- Write raw yaml files (e.g. a statefulset yaml with some auxiliary files) to
  deploy TiKV servers in Kubernetes
- Deploy with the help of helm charts

The first approach is complex and not reusable. The second approach can simplify
the deployment by abstracting away Kubernetes complexities and helm charts can
be distributed and reused. However, both approaches can't solve many operational
problems, e.g. scaling, failover (create a replacement for the failed store),
or perform custom tasks if some events happen.

## Unresolved Questions

- Detailed roadmap of tikv-operator project
