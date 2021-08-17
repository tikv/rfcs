# Record history hotspot information in TiDB_HOT_REGIONS_HISTORY

- RFC PR: https://github.com/tikv/rfcs/pull/0070
- Tracking Issue: https://github.com/pingcap/tidb/issues/25281
- Proposal Documention: https://docs.google.com/document/d/1KqjYBWhJ-YlupyDvBlObkCl-9P8WyUPL6VgYpO-LRiw/

## Summary

Create a new table `TIDB_HOT_REGIONS_HISTORY` in the `INFORMATION_SCHEMAT` schema to retrieve history hotspot regions stored by PD periodically.  

## Motivation

TiDB has a memory table `TIDB_HOT_REGIONS` that provides information about hotspot regions. But it only shows the current hotspot information. This leads to the fact that when DBAs want to query historical hotspot details, they have no way to find the corresponding hotspot information.

## Background
According to the [documentation](https://docs.pingcap.com/tidb/stable/information-schema-tidb-hot-regions) for the current `TIDB_HOT_REGIONS` table,  it can only provides information about recent hotspot regions calculated by PD according to the heartbeat from tikv.  It is inconvenient to obtain hotspot regions of past time, and locate which store the region is. For ease of use, we can store extened hotspot region infomation in PD. The DBA can query hotspot regions within a specified period in such statement in TiDB:

```SQL
SElECT * FROM information_schema.tidb_hot_regions_history WHERE update_time>='2019-04-01 00:00:00' and update_time<='2019-04-01 00:01:00';
```

The result is shown below:

```sql
+---------------------+---------+------------+----------+------------+----------+-----------+----------+---------+-------+------------+------------+------------+------------+
| UPDATE_TIME         | DB_NAME | TABLE_NAME | TABLE_ID | INDEX_NAME | INDEX_ID | REGION_ID | STORE_ID | PEER_ID | TYPE  | HOT_DEGREE | FLOW_BYTES | KEY_RATE   | QUERY_RATE |
+---------------------+---------+------------+----------+------------+----------+-----------+----------+---------+-------+------------+------------+------------+------------+
| 2019/04/01 00:00:00 | MYSQL   | USER       |        5 | PRIMARY    |        1 |         5 |        6 |      50 | READ  |         23 |  86.264363 |  70.001240 | 37.675487  |
+---------------------+---------+------------+----------+------------+----------+-----------+----------+---------+-------+------------+------------+------------+------------+
```

## Detailed design
### Existing TIDB_HOT_REGIONS

Before we introduce `TIDB_HOT_REGIONS_HISTORY`, let's see how `TIDB_HOT_REGIONS` works.

```SQL
+----------------+-------------+------+------+---------+-------+
| Field          | Type        | Null | Key  | Default | Extra |
+----------------+-------------+------+------+---------+-------+
| TABLE_ID       | bigint(21)  | YES  |      | NULL    |       |
| INDEX_ID       | bigint(21)  | YES  |      | NULL    |       |
| DB_NAME        | varchar(64) | YES  |      | NULL    |       |
| TABLE_NAME     | varchar(64) | YES  |      | NULL    |       |
| INDEX_NAME     | varchar(64) | YES  |      | NULL    |       |
| REGION_ID      | bigint(21)  | YES  |      | NULL    |       |
| TYPE           | varchar(64) | YES  |      | NULL    |       |
| MAX_HOT_DEGREE | bigint(21)  | YES  |      | NULL    |       |
| REGION_COUNT   | bigint(21)  | YES  |      | NULL    |       |
| FLOW_BYTES     | bigint(21)  | YES  |      | NULL    |       |
+----------------+-------------+------+------+---------+-------+
10 rows in set (0.00 sec)
```
There are two types of hotspot regions: `read` and `write`. The memory table retriever processes the following steps to fetch current hotspot regions from PD server:

1. TiDB send an HTTP request to the PD to obtain the hotspot regions information of the current cluster. 
   
1. PD returns the following fields：

    ```go
      // HotPeerStatShow records the hot region statistics for output
    type HotPeerStatShow struct {
      StoreID        uint64    `json:"store_id"`
      RegionID       uint64    `json:"region_id"`
      HotDegree      int       `json:"hot_degree"`
      ByteRate       float64   `json:"flow_bytes"`
      KeyRate        float64   `json:"flow_keys"`
      QueryRate      float64   `json:"flow_query"`
      AntiCount      int       `json:"anti_count"`
      LastUpdateTime time.Time `json:"last_update_time"`
    }
    ```

1. After TiDB catch the response, it  fetch the `START_KEY` and `END_KEY` of the hot region from region cache or PD by `REGION_ID` to decode the corresponding schema information like：`DB_NAME`, `TABLE_NAME`,  `TABLE_ID`,  `INDEX_NAME`, `INDEX_ID`.

1. TiDB return the hotspot region row to the upper caller.

In addition, hot regions can also be obtained directly through [pd-ctl](https://docs.pingcap.com/zh/tidb/stable/pd-control#health).  

### Table Header Design
1. New table header
  ```SQL
  > USE information_schema;
  > DESC tidb_hot_regions_history;
  +-------------+-------------+------+------+---------+-------+
  | Field       | Type        | Null | Key  | Default | Extra |
  +-------------+-------------+------+------+---------+-------+
  | UPDATE_TIME | varchar(64) | YES  |      | NULL    |       | // new
  | DB_NAME     | varchar(64) | YES  |      | NULL    |       |
  | TABLE_NAME  | varchar(64) | YES  |      | NULL    |       |
  | TABLE_ID    | bigint(21)  | YES  |      | NULL    |       |
  | INDEX_NAME  | varchar(64) | YES  |      | NULL    |       |
  | INDEX_ID    | bigint(21)  | YES  |      | NULL    |       |
  | REGION_ID   | bigint(21)  | YES  |      | NULL    |       | 
  | STORE_ID    | bigint(21)  | YES  |      | NULL    |       | // new
  | PEER_ID     | bigint(21)  | YES  |      | NULL    |       | // new
  | TYPE        | varchar(64) | YES  |      | NULL    |       |
  | HOT_DEGREE  | bigint(21)  | YES  |      | NULL    |       | // rename to HOT_DEGREE
  | FLOW_BYTES  | double      | YES  |      | NULL    |       |
  | KEY_RATE    | double      | YES  |      | NULL    |       | // new
  | QUERY_RATE  | double      | YES  |      | NULL    |       | // new
  +-------------+-------------+------+------+---------+-------+
  | REGION_COUNT| bigint(21)  | YES  |      | NULL    |       | // Deleted fields
  ```
  * Add `UPDATE_TIME` to support history.
  * Add `STORE_ID` and `PEER_ID`to track the machine of region.
  * Rename `MAX_HOT_DEGREE`  to `HOT_DEGREE` for precise meaning in history scenario.
  * Add `KEY_RATE` and `QUERY_RATE` for future expansion in hotspot determination dimensions.
  * Remove `REGION_COUNT` for disuse and repeat with `STORE_ID`.

2. Data size estimation 

    one record size: 8 * 8B(bitint) + 1 * 8B(datetime) + 4 * 64B(varchar(64)) = 328B
    Below table show data size per day and per month in 5,10,15 minutes record interval respectively given the maximum number of hotspot regions  is 1000:

  | Record Length (B)| Time Interval (Min)  | Data Size Per Day (MB)   | Data Size Per Month (MB)   |
  | ---------------- | -------------------- | ------------------------ | -------------------------- |
  | 328              | 5                    | 90.08789063              | 2702.636719                |
  | 328              | 10                   | 45.04394531              | 1351.318359                |
  | 328              | 15                   | 30.02929688              | 900.8789063                |

### Design in PD
1. Timing write：

     The leader of PD will periodically encrypt  `START_KEY` and `END_KEY`  in data,and write hotspot region data into `LevelDB`.The write interval can be configured. The write fieldes are: `REGION_ID`, `TYPE`,  `HOT_DEGREE`, `FLOW_BYTES`, `KEY_RATE`, `QUERY_RATE`, `STORE_ID`, `PEER_ID`, `UPDATE_TIME`, `START_KEY`,  `END_KEY`.

2. Timing delete

   PD runs the delete check task periodically, and deletes the hotspot region data that exceeds the configured TTL time.

3. Pull interface

   PD query the data in `LevleDB` with filters pushed down, decodes the data and returns to TiDB.

4. New options

   There are two config options need to be add in PD’s `config.go`:

   * `HisHotRegionSaveInterval`:  time interval for pd to record hotspot region information, default: 15 minutes.
   * `HisHotRegionTTL`: maximum hold day for his hot region, default: 30 days.

### Design in TiDB

1. Add memory table `TIDB_HOT_REGIONS_HISTORY`：

   Create a new memory table `TIDB_HOT_REGIONS_HISTORY` with fileds discussed above in `INFORMATION_SCHEMA` schema.

1. Add `HotRegionsHistoryTableExtractor` to push down some predicates to PD in order to  reduce network IO.

   ```go
   type HistoryHotRegionsRequest struct {
   	StartTime      int64    `json:"start_time,omitempty"`
   	EndTime        int64    `json:"end_time,omitempty"`
   	RegionIDs      []uint64 `json:"region_ids,omitempty"`
   	StoreIDs       []uint64 `json:"store_ids,omitempty"`
   	PeerIDs        []uint64 `json:"peer_ids,omitempty"`
   	HotRegionTypes []string `json:"hot_region_types,omitempty"`
   	LowHotDegree   int64    `json:"low_hot_degree,omitempty"`
   	HighHotDegree  int64    `json:"high_hot_degree,omitempty"`
   	LowFlowBytes   float64  `json:"low_flow_bytes,omitempty"`
   	HighFlowBytes  float64  `json:"high_flow_bytes,omitempty"`
   	LowKeyRate     float64  `json:"low_key_rate,omitempty"`
   	HighKeyRate    float64  `json:"high_key_rate,omitempty"`
   	LowQueryRate   float64  `json:"low_query_rate,omitempty"`
   	HighQueryRate  float64  `json:"high_query_rate,omitempty"`
   }
   ```
   
1. Add `hotRegionsHistoryRetriver` to fetch hotspot regions from all  PD servers by HTTP request,  then supplement fields like `DB_NAME`, `TABLE_NAME`,  `TABLE_ID`,  `INDEX_NAME`, `INDEX_ID` according to the `START_KEY` and `END_KEY`of the hotspot region, and merge the results.

## DrawBack

* TiDB can not add fields for deleted Table or Schema.

## Alternatives

Write to TiDB like a normal table.
* Reuse complete push-down function.
* Data is written into TiKV to provide disaster tolerance.
* The `Owner` election solves the problem of who pulls data from PD.
* Can not support scenarios using TiKV independently.

**Write solution**

1. Create a table in the mysql library:
   * advantage:
     * Reuse complete push-down function.
     * Insert, query and delete can reuse the existing functions of Tidb.
   * disadvantage
     * The content in `INFORMATION_SCHEMA` is stored in the mysql library, which feels strange.
2. Create a table in `INFORMATION_SCHEMA`:

   * advantage:
     * No need to change the query entry.
     * It is more unified in design in this library.
   * disadvantage:
     * The creation of the `init()` function itself involves a lot and is difficult to transform.

## Unresolved questions

None.

