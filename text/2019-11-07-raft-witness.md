# Raft Witness

## Summary

This RFC proposes a new role Witness to Raft, and describes how it could benefit us.

## Motivation

To tolerate 2 replicas fail, Raft needs 5 replicas, which requires a huge energy
footprint. However, with witness Raft could only need 4 or even 3 replicas to
tolerate 2 fail, and the price is latency becomes higher if some peers are
isolated, or need more time to recover from 2 peers fail. Considering they are not
common cases, we can introduce Witness into Raft.

## Dtailed Design

### The Original Article

Witness is firstly described in this [article][reduce-the-energy-footprint], and is
already implemented in [dragonboat](https://github.com/lni/dragonboat). According to
the article, Witness is like normal Voter in Raft, except

* Witness can't start new elections;
* Leader only replicates metadata (term, index) to Witness without real entries.

So Witness can reduce network traffic and storage usage in a Raft group.

However, there are some issues about about the design. For example (described in the article),
suppose there is a Raft group with Voters `A, B, C, D` and Witness `E`. `A, B, E` can
hold a quorum to commit some entries. But if `A, B` fail, `C, D, E` can't elect a new
leader forever because `E` will never start new elections, and it will always reject
vote messages from `C, D` because it has a higher *index* or *term*. Even if we can
make `C` or `D` become leader manually (with some external tools), entries committed by
`A, B, E` are still lost, which is fatal for a database.

### Improved

Considering the issues, this RFC trys to improve the machanism to make it more suitable
for distribution databases. The design is,

* Witness itself acts totally same as normal Voters;
* Witness votes voters if they have same index and term;
* Generally, Leader never appends entries to Witness;
* In some cases, Leader can append entries (NOT ONLY metadata) to Witness.

The core principle is leader only try its best to avoid appending to Witness. For example,
suppose there is a Raft group with voters `A, B` and witness `C`, and `A` is the leader.
When A broadcasts new entries to the group, it can ignore the witness `C` if `B` is recently
active.

To show what it benefits Raft, here are 2 examples:

The first is a Raft group with normal voters `A, B, C, D` and a witness `E`. It
reduces 20% energy footprint and network traffic from the topology with 5 normal voters,
but offers almost same availability: If `A, B` fail, `C` or `D` can still be elected
as the new leader, and the new leader must contains all committed entries. It's especially
useful for topology "3 centers in 2 place" if the third center doesn't need to handle
requests locally.

The second is a Raft group with normal voters `A, B, C` and witnesses `D, E`. It
reduces 40% energy footprint and network traffic from the topology with 5 normal voters.
And if `A, B` fail, C can still be elected as new leader, and then handles read requests
as normal. However new writes won't be handled until `D` and `E` catch up enough raft logs,
which means the Raft group takes more time to recover from 2 node failure.

### Swtich between Witness and Voter

As described above, Witness handles appended entries just like normal voters. Although
leader only appends entries to a Witness when too many voters are not active, Witness
needs to cleanup its local data after those voters become active again. It's easily to
do with an external timer, so we don't need to cover it in Raft.

## Drawbacks

* Write latency could increase if some voters stall.
  In the second example above, the write latency depends on the slowest peer in `A, B, C`,
  however, in the topology with 5 normal voters, it depends on the fastest 3 peers. But
  because Witness reduces 40% energy footprint, it could not be a problem in practice.
* Recovery is slow if some voters fail.
  Under the topology with 5 normal voters, the Raft group needs about 2 election timeout
  to recover from 2 node failure. But in the second example above, it needs all Witnesses
  catch up logs to be available again for writes. It could be a long time in a multi Raft
  system.

[reduce-the-energy-footprint]: https://www.researchgate.net/publication/280091830_Reducing_the_Energy_Footprint_of_a_Distributed_Consensus_Algorithm.
