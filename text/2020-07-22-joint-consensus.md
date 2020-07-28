# Use joint consensus

## Summary

To support safe conf change across data center, we need to use joint consensus.
This RFC describes how to achieve it.

## Motivation

Current approach of single step is known to be not safe when replacing a peer
within the same node or same DC. Take (a, b, c) as an example, to replacing c
by d step by step, it has to add d and then remove c. At some point the
configuration becomes (a, b, c, d). If c and d are on the same node or in the
same DC, crash of single node or single DC can make the region unavailable even
there are still two peers up.

## Detailed design

The design and implementation of joint consensus algorithm is not included in
this RFC. It should be covered by raft paper and the design in
[etcd-io/etcd#7625](https://github.com/etcd-io/etcd/issues/7625). And the
implementation has been ported to our fork tikv/raft-rs.

This RFC focuses on how to use and test raft-rs's support of joint consensus in
TiKV.

### Protocol

To keep compatibility, we need to define a new command to utilize joint
consensus. We can name it `ConfChange` or `ChangePeerV2`. The change should
look like

```protobuf
message ChangePeerV2Request {
    repeated ChangePeerRequest changes = 1;
}

message ChangePeerResponse {
    metapb.Region region = 1;
}
```

Because now that we use two phases to commit a confchange, so we need to store
joint state. Giving that we already use `is_learner` flag to indicate its role,
we can reuse the flag. According to [protocol updating rules][1], it's OK to
change a bool field to an enum field. So we can define the new peer meta as
follow:

```protobuf
enum PeerRole {
    // Voter -> Voter
    Voter = 0;
    // Learner/None -> Learner
    Learner = 1;
    // Learner/None -> Voter
    IncomingVoter = 2;
    // Voter -> Learner
    DemotingVoter = 3;
    // We forbid Voter -> None, it can introduce unavailability as discussed in
    // etcd-io/etcd#7625
    // Learner -> None can be apply directly, doesn't need to be stored as
    // joint state.
}

message Peer {
    uint64 id = 1;
    uint64 store_id = 2;
    PeerRole role = 3;
}
```

### PD

PD should not propose ChangePeerV2 until all nodes are upgraded to supporting
versions.

When using joint consensus to remove a Voter, PD should make it into a learner
first, and then remove it as discussed in [etcd-io/etcd#7625](https://github.com/etcd-io/etcd/issues/7625).

PD can use both `ChangePeer` and `ChangePeerV2` at the same time. When using
`ChangePeerV2`, it's also required to propose an empty `ChangePeerV2` to exit
joint state. The rule can be summarized as if any peer in the region is neither
voter nor learner, PD should propose an empty `ChangePeerV2` request.

It's recommended to use `ChangePeerV2` only for replacing peers for now.

### TiKV

When TiKV starts, it needs to recreate `ConfState` according to peers in
region. More specifically,

1. if all peers' roles are either `Voter` or `Learner`, they should be put into
    `voters` and `learners` respectively.
1. Otherwise peers with role `Voter` should be put into both `voters` and
    `voters_outgoing`; peers with role `Learner` should be put into `learners`;
    peers with role `IncomingVoter` should be put into `voters` only; peers
    with role `DemotingVoter` should be put into both `voters_outgoing` and
    `learners_next`.

`ChangePeer` request should be proposed and applied as usual.

For `ChangePeerV2` request, we can check the following table:
| Initial Role | AddNode | AddLearnerNode | RemoveNode
:-: | -----------: | -----------: | -----------:
None | IncomingVoter | Learner | Ignored
Voter | Voter | DemotingVoter | Illegal
Learner | IncomingVoter | Learner | None
IncomingVoter | Illegal | Illegal | Illegal
DemotingVoter | Illegal | Illegal | Illegal

`None` means not exist or removing directly. Illegal means rejecting during
propose and panicking during apply.

When proposing `ChangePeerV2`, leader should ensure the request is legal
besides a healthy quorum. `ChangePeerV2` should be transformed to
`ConfChangeV2` before proposing. The format of the two messages are similar
except `transition`. The `transition` field should always be set to
`ConfChangeTransition::Explicit`. Note even a `ChangePeerV2` has only single
change, leader should still propose it as `ConfChangeV2`, because applying
`AddLearnerNode` on a Voter is illegal in `ChangePeer` command. An empty
`ChangePeerV2` means exit joint state. It should be transformed into an empty
`ConfChangeV2`.

When applying `ChangePeerV2`, updates the role in region following the above
table. If `ChangePeerV2` is empty, then change all the `IncomingVoter` to
`Voter`, all the `DemotingVoter` to `Learner`. Region's `conf_ver` should be
increased by the count of the changes.

After applying `ChangePeerV2`, conf state should be regenerated from region
following the rules listed in the start of the section. The generated conf
state should be asserted equality with the value returned by
`RawNode::apply_conf_change`.

### TiFlash

TiFlash should work the similar way as TiKV.

### Client Routing

What's change to client is the role of a peer. What needs to be changed depends
 on how they reads `is_learner` flags. Generally, `DemotingVoter` is the nodes
 that goes to sunset, requests should not be routed to the node.

### FAQ

1. Why not use `ConfChangeTransition::Auto` or `ConfChangeTransition::Implicit`?

    The two modes will make raft propose an empty `ConfChangeV2` to exit joint
    state automatically.  However, it also adds special case into applying
    `ConfChangeV2`.

    - Apply worker needs to check if it's empty and constructs a special
        `ChangePeerV2` command either with current epoch or special mark to
        skip epoch check.
    - The empty `ConfChangeV2` is proposed during apply. Raftstore needs to
        check logs carefully to not miss the proposal and updates related
        metadata like `has_ready`.
    - It relies on raft can detect joint state and propose the command
        reliably. It may not be the case if more constraints (like flow
        control) are added. A retry mechanism is more reliable. And retry by PD
        is easier then TiKV, as `ChangeRequest` has been working the similar
        way already.

1. Why not define a new flag instead of reusing the tag number of `is_learner`?

    To keep compatibility easily. Regions with `is_learner` flag set can be
    deserialized directly without special handles.

## Drawbacks

It certainly makes the system more complicated.

## Alternatives

None.

## Unresolved questions

None.

[1]: https://developers.google.com/protocol-buffers/docs/proto3#updating
