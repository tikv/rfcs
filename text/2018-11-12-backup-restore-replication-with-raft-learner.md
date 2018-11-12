# Summary

Provide a better way to let TiKV support data backup/restore directly, and also support real-time replication.

# Motivation

Now if we want to back up the whole cluster, we have no way but through TiDB. We can use `mydumper` or other tools to dump the data from TiDB, and then use `myloader` or `lightning` to restore a new cluster. This works well for a long time but still has some problems:

1. Using the dump tools can cause a high pressure on the current running cluster. We will send the `scan` request to all region leaders and scan the data aggressively, this will cause a high I/O on the node and affect other requests.
2. If the cluster has huge data, we can’t use the common tool to dump all data to one machine. 
3. It can’t work directly for TiKV. As a CNCF project, more and more users will use TiKV directly, but now we have no way to do the backup/restore for TiKV. 
4. It can’t support real-time replication, although we can use `binlog` for TiDB, there is nothing for TiKV. We also need this.

We need to solve these problems. Fortunately, now we have already supported Raft Learner, we can use this way to create a backup node, synchronize data through Raft snapshot and replication directly. 

# Detailed design

## Learner Replication

Alongside the current running nodes, we can add a Backup node. The Backup node needs to ask PD to add learners of all regions to it. 

After a learner is added, the learner will first receive a snapshot and apply it, then replicate the Raft logs from the leader directly.

```
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

There may be a problem when the Backup first added to the cluster. Many snapshots will be created and this will also cause a high load for the Raft leaders. We can alleviate this problem below:

1. Let the PD limit the speed. 
2. Introduce the [Follower Snapshot](https://github.com/pingcap/raft-rs/issues/135) mechanism. When a new learner is added, the leader won’t send a snapshot to the learner, instead, the leader will ask a follower to generate the snapshot and send the snapshot to the new learner. After the new learner receives the snapshot and applies it, the leader will replicate the logs to the new learner again.
3. Furthermore, we can even introduce [Follower replication](https://github.com/pingcap/raft-rs/issues/136) to let the follower do the log replication directly - to reduce the leader pressure. 

Unlike common Raft peers, the learner in the Backup node doesn't apply the Raft logs, it only saves some region information in the local RocksDB. The snapshot files and Raft logs will be saved to the distributed file system like Ceph or object storage like S3,

**In the following example, we will still use Leader for snapshot and replication, and use Ceph to represent the remote storage.**

When a new learner of region 1 is added to the Backup node, it will:

1. Create a Raft peer and save the region meta in the local RocksDB.
2. Receive the snapshot from the leader and save it to the Ceph.
3. Persist the Raft state (like applied index, committed Index) in the local RocksDB. Of course, we must also record how to find the snapshot file in Ceph :-)
4. Receive the Raft logs from the leader, append them to the Ceph. For simplicity, we can use one file for one region and append the logs to it. To ensure higher safety and to avoid our foolish mistakes like writing wrong data to the state machine, we need to keep Raft logs as more as possible, maybe keep all logs and don’t compact them. 
5. Apply the committed logs. Unlike the common way which writes all data to the state machine, here we just ignore most of the commands, only care some Admin commands like Split, Merge and ConfChange.
    + The major reason that we don’t want to apply data to the state machine is performance. Unlike sequence write of Raft logs, these writes maybe random and definitely will cause a high pressure for the storage.
    + For Split, we also need to create the new split region
    + For Merge, we also need to destroy the merged region.
    + For CompactLog, we can ignore this command directly
    + For ConfChange removing the learner in the Backup node, we also need to destroy the region. 
6. Every day(or other regular times), the learner can request a new snapshot from the leader, this can speed up later Backup and Restore. 


All look fine, but there is a problem for step 5. We need to handle some corner cases for the Admin commands like Split. I will discuss it in the Restore section later. 

For better performance, we can cache the Raft logs locally and then write them to Ceph in batch.

## Backup

If we want to do the backup, we can send a Checkpoint command to the Backup node, the node will do:

1. Let all the learner sync the latest committed logs from leaders. This can be done by Follower read mechanism.
2. Get a snapshot of the current node. Because the Backup node uses RocksDB, we can easily support it. In the snapshot, we can know current all learners’ metadata, the associated snapshot files, the Raft logs, etc.

For the Backup, we only need to retrieve the metadata, it is lightweight, so we can do the backup frequently. 

## Restore

After Backup, we can get a snapshot of the whole cluster, we can use it for restore. 

For a region, if we get a snapshot file and its following Raft logs, we can first apply the snapshot, and then apply the Raft logs in order, and the region will have the consistent data eventually. We name this flow Replay later.

But here we must consider the Replay order of different regions. For example, we have region 1 and 2, and 2 is created through a Spit command in the logs of region 1 at index 100. 

The origin range of region 1 is [a, c), after Split, the range is [b, c) and region 2’s is [a, b). 

```
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

When we do the checkpoint, the checkpoint has already contained region 1 and region 2. and region 1 will be replayed from the Snapshot, region 2 will be replayed from the first log. But here is a problem when we replay region 2 before region 1. 

Assume the command in the log 1 of region 2 is `SET b = 10`, but the command in the log 11 of region 1 is `SET b = 9`. So if region 2 is replayed first, the `b` will have a wrong value. To solve this problem, we need to build a Replay order, the implementation may be easy - just to record the event.

For Split 1 -> 2 + 1, region 2 must record a event that `region 2 is split from region 1 at index 100`. When region 2 wants to replay, it finds that - “oh, region 1’s hasn’t applied to 100 yet, so let’s wait”.

For merge 1 + 2 -> 1, region 1 must also persist the similar event, and it must wait for region 2 replaying then start to replay later.

Using this way has another benefit, now we must sync the state machine forcibly when we meet Split, Merge, or other Admin commands because we have no way to recover the cluster with only replaying Raft logs. Sync can cause latency increased sometimes, so removing this can help us increase the performance.

So for restore, we need to first build the Replay tree (of course, there must be some problems when we start to do this) and replay the regions concurrently or in order. This maybe similar with lightning, we can create the region directly in the new cluster, ingest the snapshot to that region, and then add the following logs. After doing this, the region can itself apply the logs and reach a consistency. For better performance, we can let the new leader fetch the snapshot and following Raft logs from Ceph directly. 

There will be another RFC to discuss how to recover the cluster just from Raft logs. 

## Node HA

If the Backup crashed and can’t be recovered, we may lose the backup data. A simple way is to add a new Backup node directly.

Another way is to back up the metadata in Ceph regularly and let the new Backup node resume the replication easily. 

# Drawbacks

As you can see, there is one Backup node in the design, this may become the bottleneck. The snapshot transportation or log replication may occupy the whole network of this node. Another problem is that can the single node handle too many regions? 

To solve this problem, maybe we can use multi Backup nodes, each node is responsible for multi-region. For example, if TiDB is running on TiKV, we can use different Backup nodes for different databases.
Alternatives

Another simple way to support Backup and Restore may send a Checkpoint command to all TiKV nodes, each TiKV node will generate a snapshot, and then we send a Scan command to the Raft leaders and let them to scan the data of the snapshot. After we get all the data of regions, we can use lightning to restore the cluster. 

I think this can work well but has some problems:

1. This will cause a high load for the current running cluster. And if we want to limit the backup speed, the backup time may take a long time which definitely increases the pressure of RocksDB because of too old snapshot.
2. We can’t do the backup frequently. But for the above Backup node way, we only need to save the metadata of the Backup node, which is more lightweight. 
3. We still can’t keep all the Raft logs, but now for some critical production environment (like core-banking system), we need to keep the Raft logs as long as possible. 

## Unresolved questions

We may meet uncompleted transactions when recovering from the backup. For example, if we start a transaction setting `a = 1` and `b = 2`, a is in region 1 and b is in region 2. We may meet the following scenario:

```
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

If we do the backup and restore from the checkpoint, we can’t know whether the transaction is committed or not. As we can see, we can only know the transaction is committed or not when the checkpoint is before all Prewrite logs or after the Primary key Commit log.

We can’t keep the consistency with the origin cluster if we must handle these transactions, but I think it doesn’t matter. We must know that we use the backup to create a new cluster which is different from the origin one, we may write different values to these clusters, the data can be different, it is acceptable. So an easy way is just to rollback the undetermined transactions. 

If we only use the new cluster for Read, we can return error and let the client uses a older timestamp to read. This won’t change the data and can keep the consistency with the origin cluster. 


