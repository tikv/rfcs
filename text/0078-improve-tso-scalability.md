# Improve the Scalability of TSO Service

- RFC PR: https://github.com/tikv/rfcs/pull/78
- Tracking Issue: https://github.com/tikv/pd/issues/3149

## Summary

In testing large clusters for some users, we encountered a situation that is
extremely challenging for TSO performance. In this kind of clusters that may
have many TiDB instances, a certain number of PD clients will request TSO to
the PD leader concurrently, and this puts a lot of CPU pressure on the PD leader
because of the Go Runtime scheduling caused by the gRPC connection switch. With
the high CPU pressure, TSO requests will suffer from increasing average latency
and long-tail latency, which will hurt TiDB's QPS performance. To improve the
scalability of TSO Service, we will propose two kinds of enhancement to both TSO
client and server in this RFC to reduce the PD CPU usage and improve QPS performance.

## Motivation

As mentioned before, improvement is needed in the cluster that has a certain number
of TiDB instances, i.e PD clients, to reduce the CPU pressure of PD leader. With
better TSO handling performance in this case, TSO service won't be the potential
bottleneck of a big cluster easily and have a better scalability.

## Detailed design

### Current TSO processing flow

Before proposing specific improvements, let me give a brief overview of the current
PD client and TSO processing flow first.

For the PD client, every time it receive a TSO request from the upper level, it won't
send the TSO gRPC request to the PD leader immediately. Instead, it will collect as
many TSO requests as it can from a channel and batch them as a single request to
send to reduce the gRPC request number it actually sent. For example, at a certain
moment, PD client may receive 10 TSO requests at once, it will send just one TSO gRPC
request to the PD leader with an argument `count` inside the gRPC request body.

For the PD server, it just takes the request and return an incremental and unique TSO
request with 10 counts and return it to the client. After the PD client receives the
response, it will split the single TSO request into 10 different TSOs and return it to
the upper requester.

During the whole process flow, every PD client will maintain a gRPC stream with the PD
leader to send and receive the TSO request efficiently.

### Enhancement #1: Improve the batch effect

An intuitive improvement is to improve the batch effect of TSO, i.e. to reduce the number
of TSO gRPC requests by increasing the size of each batch with the same number of TSO requests.
With fewer TSO gRPC requests, the CPU pressure on the PD leader can be reduced directly.

We can introduce different batch strategies such as waiting for a while to fetch more TSO request
in a same interval or even more complicated one like a dynamic smallest batch size.

Predicting strategy is also an useful way to improve the batch effect. For example, the PD Client
could collect as much information as it needs such as latency and batch size in the last few minutes,
then base on these information, the PD client could calculate a suitable expected batch size to predict
the incoming TSO number, which make the batch waiting more effective.

![TSO Client Batch](https://i.imgur.com/vUgVSUI.png)

### Enhancement #2: Use proxy to reduce the stream number

However, as mentioned before, according to our pprof result, the main reason of high PD leader
CPU usage is the Go Runtime scheduling caused by the gRPC connection switch. Batch effect
improvement does not alleviate this part at the root. To reduce the connection switch,
fewer gRPC stream number is necessary. We can introduce the TSO Follower Proxy feature
to achieve this.

Every TSO request will be sent to different TSO servers (including both the PD leader and follower)
randomly by the PD client, and multiple TSO requests sent to the same TSO follower will be
batched again with the same logic as a PD client before being forwarded to the PD leader.
With this implementation, the gRPC pressure is distributed to each PD server, and for the
PD leader, if we have 50 TiDB instances and 5 PD instances, it only needs to maintain 4
stream connections with each PD follower rather than 50 stream connections with all the TiDB
servers.

![TSO Follower Proxy](https://i.imgur.com/WoB9YN9.png)

## Drawbacks

- Increase the TSO latency if the QPS is not high enough.
- Increase the CPU usage of the PD followers.
- Increase code complexity.
