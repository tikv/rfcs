# Pre-transfer leader

## Summary

[Raft][1] supports leadership transferring by leadership transfer extension.
This proposal adds an extra step to make leader transferring more smooth.

## Motivation

Leadership transferring may not always succeed. A follower can fail to start
election because of unapplied conf change or message loss. Even it starts
election and becomes leader at the end, it may not start to respond to requests
because logs from last term is still being committed and applied. In the
practice of TiKV, we observe a slow new leader can take seconds or minutes to
be active, which is a serious problem needs to be solved.

## Detailed design

To minimize the probability of failure, a leader can send a pre-request to ask
target follower whether it's suitable to become a leader. The follower must
check all the situations that can block it from becoming a good leader. If all
checks pass, follower replies to leader an ack to start actual leadership
transferring.

To avoid stale ack, pre-request should be attached with leader's current commit
index, which is also forwarded to ack. Leader can decide whether to continue by
checking the gap between ack commit index and its own current commit index.

Note that the above procedure can't prevent failures. In fact, it's nearly
impossible to ensure success given the complicated network environment and
applications. Leader should retry transferring at every heartbeat to make the
procedure more stable.

Note that raft paper doesn't require an election to retain leadership. But it's
not safe for TiKV because a lease mechanism is implemented based on raft. If
messages are reordered, it's possible that MsgTimeoutNow triggers a successful
election when the lease is still valid. Hence TiKV should start a new election
when it thinks transferring leadership fails.

All of above can be implemented outside of raft. But it's better to support it
in raft out of box as it solves common problems.

## Drawbacks

The proposal introduces an extra step which can make transferring leader a
little bit slower than before.

## Alternatives

Leader can use follower's apply index to decide whether to transfer leader
too. On one hand, apply index is an external properties of raft, it makes
things complicated if leader has to take care of follower's apply progress.
On the other hand, if more conditions to prevent a peer from campaign are
added, new properties may need to be observed and transferred.

## Unresolved questions

[1]: https://github.com/ongardie/dissertation
