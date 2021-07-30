# Record history hotspot information in TiDB_HOT_REGIONS_HISTORY

- RFC PR: https://github.com/tikv/rfcs/pull/0070
- Tracking Issue: https://github.com/pingcap/tidb/issues/25281
- Proposal Documention: https://docs.google.com/document/d/1KqjYBWhJ-YlupyDvBlObkCl-9P8WyUPL6VgYpO-LRiw/

## Summary

Create a new table `TIDB_HOT_REGIONS_HISTORY` in the `INFORMATION_SCHEMAT` database to retrieve history hotspot information stored by PD periodically.  

## Motivation

TiDB has a memory table `TIDB_HOT_REGIONS` that provides information about hotspot regions. 
But it only shows the current hotspot information. This leads to the fact that when DBAs want to query historical hotspot details, 
they have no way to find the corresponding hotspot information.

## Background
According to the [documentation](https://docs.pingcap.com/tidb/stable/information-schema-tidb-hot-regions) for the current `TIDB_HOT_REGIONS` table, 
the table provides information about hotspot regions shown in the following table: 

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
There are two types of hotspot regions: read and write. 
The memory table retriever processes the following steps to fetch current hotspot regions from PD server:

1. TiDB send an HTTP request to the PD to obtain the hotspot regions information of the current cluster. 
    `pd/api/v1/hotspot/regions/read` for read hotspot and `pd/api/v1/hotspot/regions/write` for write hotspot. 
    PD returns the following fields：
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
1. Parse response and fetch the start key and end key of the hot region from region cache or PD by `REGION_ID` to calculate the region's corresponding schema information like：`TABLE_ID`, `TABLE_NAME`, `DB_NAME`, `INDEX_ID`, `INDEX_NAME`.
1. Return the hotspot information to the upper call.

## Detailed design

### Table Header Design
1. New table header
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
  | FLOW_BYTES     | bigint(21)  | YES  |      | NULL    |       |
  // New fields
  | KEY_RATE       | bigint(21)  | YES  |      | NULL    |       |
  | QUERY_RATE     | bigint(21)  | YES  |      | NULL    |       |
  | STORE_ID       | bigint(21)  | YES  |      | NULL    |       |
  | UPDATE_TIME    | datetime    | YES  |      | NULL    |       |
  // Deleted fields
  | REGION_COUNT   | bigint(21)  | YES  |      | NULL    |       |
  +----------------+-------------+------+------+---------+-------+
  ```
  * Add `STORE_ID` to track the machine of region.
  * Add `UPDATE_TIME` to support history.
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

     The leader of PD will periodically encrypt  `start_key` and `end_key`  in data,and write hotspot region data into `LevelDB`.The write interval can be configured. The write fieldes are: `region_id`, `type`,  `max_hot_degree`, `flow_bytes`, `key_rate`, `query_rate`, `store_id`, `update_time`, `start_key`,  `end_key`.

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

   Create a new memory table `TIDB_HOT_REGIONS_HISTORY` to the existing `INFORMATION_SCHEMA` database.

1. Add pull function：

   Pull  and combine the data from all  PD servers,  and add fields like `table_id`, `index_id`, `db_name`, `table_name`, `index_name` according to the `start_key` and `end_key`  of the hotspot region.

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

