# Follower Replication

## Summary
This RFC introduces a new mechanism in Raft Protocol which allows a follower to send raft logs to other followers and learners.  The target of this feature is to reduce network transmission costs between different data centers in Log Replication.

## Motivation
In the origin raft protocol, a follower can only receive new raft logs and snapshots from the leader, which could be insufficient in some situations. For example, when a raft cluster is distributed in different data centers, log replication between a new node and the leader is expensive as they are located at different data centers. In this case, internal follower-to-follower transfer in one data center can be far more efficient than the traditionally stipulated leader-to-follower transfer.

## Detailed design
There are two key concepts of design:
- The leader must know each node’s data center in the raft cluster.
- The leader is able to ask a follower to send raft logs or a snapshot to other followers.

### The Group and Groups
We introduce a new concept *group* that represents all the raft nodes in a datacenter and any node which belongs to the group can be called a *group member*. Every raft node should have a new configuration called *Groups* which contains all the groups as the name is. For example, in a 2DC based raft cluster with 5 nodes ABCDE where A, B, and C are at DC1 and  DE are at DC2, the `Groups` might be organized like: 
```
Group1: DC1 -> [A, B, C]
Group2: DC2 -> [D, E]
```
The leader must be easy to know whether or not nodes belong to the same group or get all the group members. Since `Groups` is a configuration that will never be persistent in Storage and is volatile, a raft node exposes the ability to modify it on the fly with the benefit of flexibility on the upper layer. 

### The Delegate and Commission
As the `Groups` is configured, the leader is able to choose a group member as a *delegate* to send entries to one or the rest group members in Log Replication, which is called Follower Replication. To implement Follower Replication, we basically need three main steps:

1. The leader picks a group member as a delegate of the group
2. The leader prepares and sends some *commission*s that indicate what entries the picked delegate will send to others by a new message type MsgBroadcast.
3. The delegate receives this message and executes the commissions

Here is a diagram showing how Follower Replication works:
```
                                        +-----------+
                                        |           |
       +--------------------------------+     C     |
       |           MsgAppendResp        |           |
       |                                +-----^-----+
       |                                      |
       |                                      |  MsgAppend(e[4,5])
       |                                      |
+------v-----+                          +-----+-----+
|            |                          |           |
|   leader   +-------------------------->     B     | delegate
|            |   MsgBroadcast{          |           |
+------^-----+     e[2,5]               +-----+-----+
       |           Commission(C, [4,5])       |
       |           Commission(D, [3,5])       |  MsgAppend(e[3,5])
       |         }                            |
       |                                +-----v-----+
       |                                |           |
       +--------------------------------+     D     |
                   MsgAppendResp        |           |
                                        +-----------+

```
Note: 
- `e[x, y]` stands for all the entries within an index range [x, y] (both inclusive)
- `Commission(target, [x, y])` stands for a job that the delegate should send `e[x, y]` to the target

#### Choose a Delegate
Since a delegate is responsible to send entries to other group members, it must contain entries as much as possible to complete the commission. The leader follows a bunch of rules to choose a delegate of a group:
1. If all the members are requiring snapshots, choose the delegate randomly.
2. Among the members who are requiring new entries, choose the node satisfies conditions below :

    1. Must be `recent_active`
    2. The progress state should be `Replicate` but not `paused`
    3. The progress has the smallest `match_index`

3. If no delegate is picked, the leader does Log Replication itself. Especially, if a group contains the leader it self, no delegate will be set by default except in some cases such as massively large group, which is able to be controlled by upper layer.

And let's talk about these rules carefully:
For rule 1, It's obvious that a node requiring a snapshot is not a proper choice because its progress is too far behind and its raft logs are highly possible to be stale. 

In most cases, what `recent_active` is true means the node keeps communicating with the leader if `check_quorum` is enabled so rule 2.1 is reasonable.

As for rule 2.2, the point is that in `raft-rs` the `Progress` of a node has a flow control mechanism and the leader shouldn’t send messages to a node with `paused` Progress. And `Replicate` state indicates a node is continuously receiving raft logs from the leader, which means this node is somewhat 'healthy' in the viewpoint of the leader.

The rule 2.3 is a little bit subtle. If a node passes rule 2.1 and 2.2, we can say it’s an active node with a smooth network connection. Under these circumstances, the node with the smallest `match_index` may have the greatest chance of having enough entries to be sent to every other group member. 

If the delegate of a group is determined, it’ll be cached in memory for later usage until Groups configuration is changed or the delegate is evicted by rejection message. And the flow control should be valid for any cached delegate.

#### MsgBroadcast and Commission
After the delegate is picked, the leader will prepare a `MsgBroadcast` message which is sent to the delegate. A MsgBroadcast just looks like a `MsgAppend` or `MsgSnapshot` to be sent to the delegate, which often only carries entries the delegate needs but also includes a collection of commissions. But if some members are requiring entries and others are requiring snapshots, the `MsgBroadcast` be generated with both entries and a snapshot.
A *commission* just describes the range of entries or a snapshot that should be sent to the target. It looks like:
```
Commission {
    type  // Only two types: Append or Snapshot
    target  // The target group member
    last_index 
    log_term
}
```

You can just treat a commission as the metadata of a MsgAppend or MsgSnapshot since the `last_index` and `log_term` are set by the leader according to its progress set.

When the delegate receives a `MsgBroadcast`, it might meet any scenario below:
1. If the message declares that the delegate needs a snapshot, it means that all the group members are requiring snapshots. Every commission type should be Snapshot and the delegate just broadcasts to others.
2. If the message declares that the delegate needs entries, it first tried to append incoming entries to its raft log. As the origin raft protocol,  if the appending fails, it sends a rejecting message to the leader and then the leader will try to pick a new delegate to re-send *commission*s again.
3. If entires appending in step 2 succeeds, the delegate will try to execute all the *commission*s. And some *commission* executions could be failed due to the stale progress state in the leader node. The delegate will collect all the failed *commission*s and send them back to the leader to trigger a new broadcast message so that Log Replication is always ongoing.

And it’s significant to update the inflight of progress when the corresponding *commission* is generated and rollback the inflight once the leader receives failed *commission*s in step3 above, which also guarantees that the actual progress of group members (except the delegate) will not be stale.

## Drawbacks
When there is a large group or many groups, the log committing speed could be slow as entries will be sent to delegate first. 

## Alternatives

## Unresolved questions
The rule 2.3 for choosing a delegate might need some rethink.
