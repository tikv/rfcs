# Witness

## Background

On cloud env, TiKV is equipped with AWS EBS storage for single node which provides durability of 99.9%. Moreover, TiKV organizes 3 Raft replicas on top of that, which may be a overkill for durability. To reduce cost, we can introduce a mechanism named "2 Replicas With 1 Log Only" instead of 3 replicas to save 30% storage resources and cut down about 100%(1 core) CPU usage empirically on the cloud. Compared with the original 3 replicas architecture, the "1 Log Only" replica stores raft logs but does not apply it, which still ensures data consistency through the Raft protocol.

## Design

The overall idea is to introduce a new peer role called "witness" to differentiate with both voter and learner. Witness accounts for quorum but doesn't store full kv data, that is to say, it doesn't apply any write command to kvdb, but admin command.

### Availability and Durability

It only persists raft log to help replicate the missing log when one voter fails. With that, it provides comparable availability at the cost of lower durability.

- For availability(the SLA is 99.99% monthly referring to Service Level Agreement for TiDB Cloud Services | PingCAP)
  - it still keeps the promise for 3 replicas set-up that any one of the peer down/isolation doesn't affect the Raft group normal process, though the unavailable time is little longer
    - If the leader downs, the witness will replicate the logs to the other voter. Then the other voter could be the leader normally. It needs one round of election timeout(11s) and a little time to replicate and transfer leader which is possibly finished within 1s.
    - If the witness or the follower downs, the two left voters work well as normal case. It needs one round of election timeout(11s) on average to recover, just like before.
  
|  | SLA | promised max down time |
| :----:| :---: | :----: |
| Before | 99.99% | 3600 *24* 365 * (100% - 99.99%) = 3153.60s |
| After | (1 - 3440.58/(3600 *24* 365)) * 100% = 99.989% ≈ 99.98% | (11+1)/11 * 315360s = 3440.58s |

- For durability, durability is reduced for extreme cases for one peer down
  
|  | For one peer downs | For one peer downs and the other peer is lagging behind(need snapshot) |For two peer downs |
| :----:| :---: | :----: | :----: |
| Before | no loss | no loss (after unsafe recovery) | lose update (after unsafe recovery) |
| After | no loss | lose update (after unsafe recovery) | lose update or lose all data (after unsafe recovery) |

### Raft Process

To let the witness replicate the log to the lagged voter, the witness should be elected as the leader at that time. Some rules related to witness:

- For the election process, peers vote for witness only when their last log term/index is smaller than(not include equal) that of witness. With that, witness is elected as leader with lower possibility. Whereas, in the case of one voter failure, witness can be elected as leader if the other voter's log is out of date.
- When a witness is elected as the leader. PD scheduler would file a transfer leader to one of the healthy voters. In the transfer leader process, it will replicate the latest logs to the target leader first and then do the actual transfer.
- During the time of witness being the leader, all reads and writes are rejected with error `IsWitness` and it will be tried innerly.
Raft log GC
If there is any voter lagged behind, the log truncation won't be triggered even if it's force mode(raft log size/count exceeds the threshold or raft engine purge). If it is the witness that lags behind a lot, it's acceptable to trigger log truncation.
If there is any voter lagging behind, the log truncation of the witness shouldn't be triggered even if it's force mode(raft log size/count exceeds the threshold or raft engine purge), otherwise the witness can't help the lagging voter catch up logs when leader is down.
To solve that:
- Add a field voter_replicated_index in CompactLog admin command. When applying that command, the witness won't do an actual compact if voter_replicated_index is smaller than the compact index.
  - Compact index should be queued. If witness receives a voter_replicated_index that is larger than the pending compact index, logs can be deleted.
  - The witness should query the leader periodically whether voter_replicated_index is updated if CompactLog admin command isn't triggered for a while.
- If the voter is down or quite slow, voter_replicated_index, of course, can't be advanced any further. To avoid the witness accumulating too many logs leading to disk full,  PD should promote the witness to non-witness if there is any voter is down/pending peer.

### Conf Change

Witness as a flag

```protobuf
message Peer {
    uint64 id = 1;
    uint64 store_id = 2;
    PeerRole role = 3;
    bool   is_witness = 4;
}
```

Witness is orthogonal to peer role, so either an (incoming/demoting) voter or a learner(except TiFlash) can be a witness. Here are all possible transition paths:

- {A(voter), B(voter)} + AddLearnerNode(C(witness)) = {A(voter), B(voter), C(learner, witness)}
  - Add learner triggers a snapshot which is empty as target peer is witness. Then it appends logs from leader as normal (Modify the snapshot process to send and apply empty snapshot for witness which only updates the truncated state of the witness)
- {A(voter), B(voter), C(learner, witness)} + AddNode(C(witness)) = {A(voter), B(voter), C(voter, witness)}
  - Nothing different, just conf change learner -> voter
- {A(voter), B(voter), C(voter)} + AddNode(C(witness)) = {A(voter), B(voter), C(voter, witness)}
  - Clean up the kvdb data of region range after the conf change is applied
- {A(voter), B(voter), C(learner)} + AddLearnerNode(C(witness)) =  {A(voter), B(voter), C(learner, witness)}
  - Same as above
- {A(voter), B(voter), C(voter, witness)} + AddLearnerNode(C(witness)) = {A(voter), B(voter), C(learner, witness)}
  - Nothing different, just conf change voter -> learner
- {A(voter), B(voter), C(learner, witness)} + RemoveNode(C(witness)) = {A(voter), B(voter)}
  - Nothing different, just remove the learner peer
- {A(voter), B(voter), C(voter, witness)} + AddNode(C) = {A(voter), B(voter), C(voter)}
  - Forbidden, as it would affect availability before witness has finished applying snapshot. So we need to demote to learner first, then change to non-witness.
- {A(voter), B(voter), C(learner, witness)} + AddLearnerNode(C) = {A(voter), B(voter), C(learner)}
  - Right after the conf change is applied, call `request_snapshot()` explicitly to request a snapshot from the leader, then apply snapshot the same way as adding a new peer.
  - Only after applying the snapshot can the peer provide service. How?  
    - Reset the region local state to uninitialized when applying conf change. When uninitialized, it doesn't handle raft messages except snapshot.
    - What if there is a restart?
      - Region local state is persisted in kvdb, after restart, it's still regarded as uninitialized.
  - How does PD know this operator is finished?
    - Leader collects apply index progress of other peers(passed by the raft message extra_ctx field)
      - Witness and uninitialized peer's apply index is 0. After the witness applies the snapshot, it propagates a normal apply index.
      - Peer(except witness) is regarded as pending peer whose apply index is smaller than leader's truncated index
      - For witness -> nonwitness operator, it won't be finished until the peer is not witness and the peer is not pending anymore

### Snapshot Process

Generating, sending, applying snapshot are light-weight operations for witness, which only need to include metadata.

Add a for_witness field in snapshot meta, and set it when sending a snapshot for witness.

```protobuf
message raft_serverpb.SnapshotMeta {
    repeated SnapshotCFFile cf_files = 1;
    bool for_balance = 2;
    bool for_witness = 3;
}
```

Correspondingly, snapshot key is extended with a suffix _witness if it's for witness

### Region Split

The peer split from a witness is still a witness, just like how other peer roles do.

### Region Merge

The source and the target peers located in the same store must be of the same role, both witnesses or both voters.

### PD Schedule

- Through PD placement rule to make sure witness is added for each region when enabled
  - Only take effect when cluster version >= 6.2.0
- Rule checker
  - When witness placement rule is enabled
    - Promotes the witness to voter when region has any down/pending voter
    - Replace the witness with witness when offline a store
  - No matter whether witness placement rule is enabled or disabled (to improve availability)
    - When peer's downtime exceeds the threshold(30min), add a witness and remove the down peer. Then witness is promoted to non-witness gradually.
- Merge checker considers witness should be paired.
- Transfer leader schedule filter out witness
- Consider witness for balance scheduler
  - Balance region scheduler counts witness peer's size as 0
  - Balance region scheduler won't do any witness replacement
  - Hot scheduler still considers the witness for balancing hot peer
  - Hot scheduler filters out witness when moving follower for follower read
  - Add a balance witness count scheduler
    - Filter out hot region
TiKV Client
- Skip the witnesses when follower reads choose the target peer
- Retry on `IsWitness` error

## Failure Cases

### The leader is down and the witness has more logs
For the case illustrated below, when the previous leader — p3 goes down. If the other voter(p1)  does not have the latest commit, then witness(p2) can replicate the latest logs to p1. After voter(p1) receives the up-to-date log from witness(p2), voter(p1) is able to become the leader and serve normally. In this way, a witness minimizes durability loss as much as possible.

| log idx | 1 | 2 | 3 | 4 | 5 | 6 |
| :--:| :--: | :--: | :--: | :--: | :--: | :--: |
| p1(voter) | term1 | term2 | term2 | 
| p2(witness) | term1 | term2 | term2 | term2 | term3
| p3(voter)❌ | term1 | term2 | term2 | term2 |term3(committed) |term3 

Here is the process to recover:

1. Election timeouts and witness(p2) triggers an election.
2. The other voter(p1) vote for the witness because the witness has newer logs.
3. The witness becomes the leader.
4. The witness proposes a transfer leader command.
5. Raft transfers leader to the voter(p1) after making sure it replicates the latest log to the target leader.
6. The voter(p1) becomes the leader.
7. PD schedules a conf change which promotes the witness to voter.
8. If the voter(p3) is still being down for while, remove it and add a new witness; if the voter(p3) is back online, demote one of the voters to witness.

### The leader is down and the witness has more logs, but the log is truncated
The case can't be recovered due to the witness not having enough previous logs. But this case can't happen as GC logic prevents log truncation when a voter lags behind and snapshot prevents the
For defensive programming, if this happens, make sure the region won't panic, just keep witness as the leader but can't transfer leader.

| log idx | 1 | 2 | 3 | 4 | 5 | 6 |
| :--:| :--: | :--: | :--: | :--: | :--: | :--: |
| p1(voter) | term1 | term2 | 
| p2(witness) | (truncated) | (truncated) | (truncated) | term2 | term3
| p3(voter)❌ | (truncated) | (truncated) | (truncated) | term2 |term3(committed) |term3 

### The leader is down and the other follower has more or equal logs

| log idx | 1 | 2 | 3 | 4 | 5 | 6 |
| :--:| :--: | :--: | :--: | :--: | :--: | :--: |
| p1(voter) | term1 | term2 | term2 | term2 | term3
| p2(witness) | term1 | term2 | term2 | term2 | term3
| p3(voter)❌ | term1 | term2 | term2 | term2 |term3(committed) |term3 

Here is the process to recover:

1. Election timeouts and the other voter(p1) triggers an election.
2. The witness(p2) votes for the voter due to the voter has newer logs.(If the witness trigger the election, the other voter won't voter for it due to the witness doesn't have newer logs).  
3. The voter(p1) becomes the leader.
4. PD schedules a conf change which promotes the witness to voter.
5. If the voter(p3) is still being down for while, remove it and add a new witness; if the voter(p3) is back online, demote one of the voters to witness.

### The follower voter is down

1. PD schedules a conf change which promotes the witness to voter.
2. If the voter is still being down for while, remove it and add a new witness; if the voter is back online, demote one of the follower voters to witness.

### The witness is down

1. If the witness is still being down for while, remove it and add a new witness; if the witness is back online, do nothing.
