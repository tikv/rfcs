# Backup & Restore Replication with Raft Learner

## Summary

Provide a better way to let TiKV support data backup/restore directly, and also
support real-time replication.

## Motivation

Now if we want to back up the whole cluster, we have no way but through TiDB.
We can use `mydumper` or other tools to dump the data from TiDB, and then use
`myloader` or `lightning` to restore a new cluster. This works well for a long
time but still has some problems:

1. Using the dump tools can cause a high pressure on the current running
   cluster. We will send the `scan` request to all Region leaders and scan the
   data aggressively, which will cause a high I/O on the node and affect other
   requests.
2. If the cluster has large amounts of data, we can’t use the common tool to
   dump all data to one machine.
3. It can’t work directly for TiKV. As a CNCF project, TiKV will be directly
   used by more and more users, but now we have no way to do the backup/restore
   for TiKV.
4. It can’t support real-time replication. Although we can use `binlog` for
   TiDB, there is nothing for TiKV. We also need this.

We need to solve these problems. Fortunately, now we have already supported
Raft Learner, so we can use this way to create a backup node, synchronize data
through Raft snapshot and replication directly.

## Detailed design

### Learner Replication

Alongside the current running nodes, we can add a Backup node. The Backup node
needs to ask PD to add learners of all Regions to it.

After a learner is added, the learner will first receive a snapshot and apply
it, then replicate the Raft logs from the leader directly.

```graph
+---------+
|Node A   |
+---------+----+                          +-----+
|Region 1*|    |                     +----|Ceph |
+---------+    |      +---------+    |    +-----+
               |      |Backup   |    |
+---------+    |      +---------+    |
|Node B   |    |      |Region 1 |    |    +-----+
+---------+-----------+---------+---------|S3   |
|Region 2*|    |      |Region 2 |    |    +-----+
+---------+    |      +---------+    |
               |      |Region 3 |    |
+---------+    |      +---------+    |    +-----+
|Node C   |    |                     +--- |Disk |
+---------+----+                          +-----+
|Region 3*|
+---------+
 ```

There may be a problem when the Backup node is first added to the cluster.
Many snapshots will be created and this will also cause a high load for the
Raft leaders. We can alleviate this problem as below:

1. Let the PD limit the speed.
2. Introduce the [Follower
   Snapshot](https://github.com/pingcap/raft-rs/issues/135) mechanism. When a
   new learner is added, the leader won’t send a snapshot to the learner.
   Instead, the leader will ask a follower to generate the snapshot and send
   the snapshot to the new learner. After the new learner receives the snapshot
   and applies it, the leader will replicate the logs to the new learner again.
3. Furthermore, we can even introduce [Follower
   replication](https://github.com/pingcap/raft-rs/issues/136) to let the
   follower do the log replication directly - to reduce the leader pressure.

Unlike common Raft peers, the learner in the Backup node doesn't apply the Raft
logs, it only saves some Region information in the local RocksDB. The snapshot
files and Raft logs will be saved to the distributed file system like Ceph or
object storage like S3,

**In the following example, we will still use Leader for snapshot and
replication, and use Ceph to represent the remote storage.**

When a new learner of Region 1 is added to the Backup node, it will:

1. Create a Raft peer and save the Region meta in the local RocksDB.
2. Receive the snapshot from the leader and save it to the Ceph.
3. Persist the Raft state (like applied index, committed Index) in the local
   RocksDB. Of course, we must also record how to find the snapshot file in
   Ceph. :-)
4. Receive the Raft logs from the leader, and append them to the Ceph. For
   simplicity, we can use one file for one Region and append the logs to it. To
   ensure higher safety and to avoid our foolish mistakes like writing wrong
   data to the state machine, we need to keep Raft logs as more as possible,
   maybe keep all logs and don’t compact them.
5. Apply the committed logs. Unlike the common way which writes all data to the
   state machine, here we just ignore most of the commands, and only care some
   Admin commands like Split, Merge and ConfChange.

    + The major reason that we don’t want to apply data to the state machine is
      performance. Unlike sequential write of Raft logs, these writes maybe
      random and definitely will cause a high pressure for the storage.
    + For Split, we also need to create the new split Region
    + For Merge, we also need to destroy the merged Region.
    + For CompactLog, we can ignore this command directly.
    + For ConfChange removing the learner in the Backup node, we also need to
      destroy the Region.
6. Every day (or other regular times), the learner can request a new snapshot
   from the leader, which can speed up later Backup and Restore.

All look fine, but there is a problem for step 5. We need to handle some corner
cases for the Admin commands like Split. I will discuss it in the Restore
section later.

For better performance, we can cache the Raft logs locally and then write them
to Ceph in batches.

### Backup

If we want to do the backup, we can send a Checkpoint command to the Backup
node, and the node will do:

1. Let all the learner sync the latest committed logs from leaders. This can be
   done by Follower read mechanism.
2. Get a snapshot of the current node. Because the Backup node uses RocksDB, we
   can easily support it. In the snapshot, we can know current all learners’
   metadata, the associated snapshot files, the Raft logs, etc.

For the Backup, we only need to retrieve the metadata. It is lightweight, so we
can do the backup frequently.

### Restore

After Backup, we can get a snapshot of the whole cluster, we can use it for
restore.

For a Region, if we get a snapshot file and its following Raft logs, we can
first apply the snapshot, and then apply the Raft logs in order, and the Region
will have the consistent data eventually. We name this flow Replay later.

But here we must consider the Replay order of different Regions. For example,
we have Region 1 and 2, and 2 is created through a Spit command in the logs of
Region 1 at index 100.

The origin range of Region 1 is [a, c). After Split, the range of Region 1 is
[b, c) and Region 2 is [a, b).

```graph
                                            checkpoint
                                                |
+---------+      +---------+---+---+---+---+---+|
|Region 1 |      |Snapshot |11 |...|100|101 102||
+---------+      |Index 10 +---+---+---+---+---+|
                 +---------+         |          |
                              Split  |          |
                              1 -> 2 + 1        |
                                     |          |
                                     |          |
+---------+                          | +---+---+|
|Region 2 |                          +>|1  |2  ||
+---------+                            +---+---+|
                                                v
```

When we do the checkpoint, the checkpoint has already contained Region 1 and
Region 2. Region 1 will be replayed from the Snapshot, and Region 2 will be
replayed from the first log. But here is a problem when we replay Region 2
before Region 1.

Assume the command in the log 1 of Region 2 is `SET b = 10`, but the command in
the log 11 of Region 1 is `SET b = 9`. So if Region 2 is replayed first, the
`b` will have a wrong value. To solve this problem, we need to build a Replay
order, the implementation may be easy - just to record the event.

For Split 1 -> 2 + 1, Region 2 must record a event that `Region 2 is split from
Region 1 at index 100`. When Region 2 wants to replay, it finds that - “oh,
Region 1 hasn’t applied to 100 yet, so let’s wait”.

For merge 1 + 2 -> 1, Region 1 must also persist the similar event, and it must
wait for Region 2's replaying and then starts to replay later.

Using this way has another benefit, now we must sync the state machine forcibly
when we meet Split, Merge, or other Admin commands because we have no way to
recover the cluster with only replaying Raft logs. Sync can cause latency
increase sometimes, so removing this can help us increase the performance.

So for restore, we need to first build the Replay tree (of course, there must
be some problems when we start to do this) and replay the Regions concurrently
or in order. This may be similar to lightning. We can create the Region
directly in the new cluster, ingest the snapshot to that Region, and then add
the following logs. After doing this, the Region can itself apply the logs and
reach a consistency. For better performance, we can let the new leader fetch
the snapshot and following Raft logs from Ceph directly.

There will be another RFC to discuss how to recover the cluster just from Raft
logs.

### Node HA

If the Backup crashed and can’t be recovered, we may lose the backup data. A
simple way is to add a new Backup node directly.

Another way is to back up the metadata in Ceph regularly and let the new Backup
node resume the replication easily.

## Drawbacks

As you can see, there is one Backup node in the design, this may become the
bottleneck. The snapshot transportation or log replication may occupy the whole
network of this node. Another problem is whether the single node can handle too
many Regions.

To solve this problem, maybe we can use multi Backup nodes, each node is
responsible for multi-region. For example, if TiDB is running on TiKV, we can
use different Backup nodes for different databases.

## Alternatives

Another simple way to support Backup and Restore may be sending a Checkpoint
command to all TiKV nodes. Each TiKV node will generate a snapshot, and then we
send a Scan command to the Raft leaders and let them scan the data of the
snapshot. After we get all the data of Regions, we can use lightning to restore
the cluster.

I think this can work well but has some problems:

1. This will cause a high load for the current running cluster. And if we want
   to limit the backup speed, the backup time may take a long time which
   definitely increases the pressure of RocksDB because of too old snapshot.
2. We can’t do the backup frequently. But for the above Backup node way, we
   only need to save the metadata of the Backup node, which is more
   lightweight.
3. We still can’t keep all the Raft logs, but now for some critical production
   environment (like core-banking system), we need to keep the Raft logs as
   long as possible.

### Unresolved questions

We may meet uncompleted transactions when recovering from the backup. For
example, if we start a transaction setting `a = 1` and `b = 2`, a is in Region
1 and b is in Region 2. We may meet the following scenario:

```244
                             checkpoint
                                 |
+--------+    +----------------+ | +--------------+
|Region 1|    |       101      | | |     102      |
+--------+    | Prewrite a = 1 | | | commit a = 1 |
              +----------------+ | +--------------+
+--------+    +----------------+ | +--------------+
|Region 2|    |       101      | | |     102      |
+--------+    | Prewrite b = 2 | | | commit b = 2 |
              +----------------+ | +--------------+
                                 v
```

If we do the backup and restore from the checkpoint, we can’t know whether the
transaction is committed or not. As we can see, we can only know the
transaction is committed or not when the checkpoint is before all Prewrite logs
or after the Primary key Commit log.

We can’t keep the consistency with the origin cluster if we must handle these
transactions, but I think it doesn’t matter. We must know that when we use the
backup to create a new cluster that is different from the origin one, we may
write different values to these clusters, and the data can be different, which
is acceptable. So an easy way is just to roll back the undetermined
transactions.

If we only use the new cluster for Read, we can return an error and let the
client use an older timestamp to read. This won’t change the data and can keep
the consistency with the origin cluster.
