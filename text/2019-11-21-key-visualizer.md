# RFC: KeyVisualizer: The visualized hotspot profiler

## Summary

Add an enabled-by-default `KeyVisualizer` component to PD, providing
a visual webpage to show key heatmap of the `TiKV` cluster, which is
useful for troubleshooting and reasoning inefficient application
usage patterns.
`KeyVisualizer` is accessible in-browser via the URL from PD.

## Motivation

At present, if someone wants to troubleshoot a performance issue in `TiKV`,
he will usually make use of some existing diagnostic tools, such as `pd-ctl`,
`Prometheus` and `Grafana`, which are hard to learn, intuitive, not
 efficient, unable to recognize the pattern of application and hard to
 make correct judgment basing on them.

`Google BigTable` has similar problems. So they provided a heatmap profiler called
[`BigTable KeyVisualizer`](https://cloud.google.com/bigtable/docs/keyvis-overview)
([Video](https://youtu.be/aKJlghIygQw)), which is recommended that the user
first use to troubleshoot performance problems. After investigation, we realized
that it has good features that also suits `TiKV`, thus we added some of those
features to our design.

## Detailed Design

### Features

`KeyVisualizer` is enabled by default. It collects various statistics information
in regular intervals, and renders multiple heatmaps in among the last hour/24 hours/week/month.

#### Frontend Features

The heatmap is a two-dimensional color chart where:

- X-axis: time axis
- Y-axis: key axis
- Color: value

`KeyVisualizer` provides several different heatmaps for:

- Read traffic
- Read Ops (long-term plan)
- Write traffic
- Write Ops (long-term plan)
- Read latency (long-term plan)
- Write latency (long-term plan)
- CPU usage (long-term plan)
- Memory usage (long-term plan)
- Disk read and write (long-term plan)
- Number of client connections (long-term plan)

Interactions:

- Display label axis showing DB name, table name, and row id of the current row
- When hover, display a popup containing the key range,
    time range and all known statistics of the cell under the cursor
- Warn abnormal area (long-term plan):
- Traffic exceeds threshold
  - Latency or wait time is too long
  - Brush select or click on a label to zoom

### Backend Features

- Extend `PD` API to provide necessary data for frontend
- Persist statistics data
- ​​Support decoding various key encoding (default `TiDB` encoding)

### `PD` Frontend Implementation

#### Retrieve Data

The frontend gets heatmap data via the new PD API.

The API interface is defined as:

URL:  `/pd/apis/keyvisual/v1/heatmaps?`

Request parameters:

- `type`: indicating which heatmap to show
  - `write_bytes`
  - `read_bytes`
  - `write_keys`
  - `read_keys`
  - `startkey`: the start of the heatmap key axis
  - `endkey`:  the end of the heatmap key axis
  - `starttime`: the start of the heatmap time axis (UNIX timestamp in second)
  - `endtime`: the end of the time axis of the heatmap (UNIX timestamp in second)

Respond data:

```typescript
export type KeyAxisEntry = {
  key: string
  labels: string[]
}

export type HeatmapData = {
  timeAxis: number[]
  keyAxis: KeyAxisEntry[]
  values: number[][]
}
```

#### Render the Heatmap

Render the heatmap once data retrieved.

- Use `D3.js` to support the time axis, the key axis, and the Label axis.
- Use `OffscreenCanvas` for background rendering,
    when each pixel is corresponding to a data cell.
    At least 1000 x 1000 cells can be displayed in one page.

POC work:

> workload:
> sysbench oltp_update_non_index -table-size=4

![key-visualizer-poc](../media/key-visualizer-poc.png)

### PD Backend Implementation

#### Collect Region Statistics

Retrieve statistics data among all regions per miniute from `PD Server`.

Unresolved problems:

- The number of regions can reach up to millions.
    Some effort should be made on time and space consumption.
- Due to the splitting and merging of the Region,
    and the information is reported asynchronously, some ranges of keys may be empty.

#### Compression of Region Statistics

##### Compression of Time Axis

To reduce the memory overhead, configurable segmentation compaction is used for the
time-axis. Currently divided into 3 layers:

- The first layer holds the data in the last day,
    where each column has a time delta of 1 minute,
    which generates 60 x 24 = 1440 ticks.
- The second layer holds the data in the last week ,
    where each column has a time delta of 15 minute,
    which generates 7 x 24 x 4 = 672 ticks.
- The third layer holds the data in the last month ,
    where each data point has a time delta of 1 hour,
    which generates 30 x 24 = 720 ticks.

Up to a total of 2872 time ticks can be shown at the same time.

##### Compaction of Key-Axis

`Regions` can reach millions of orders, so reporting the statistics for each row
is not realistic. Data needs to be aggregated according to certain rules to
reduce the amount of data. There are two strategies available
(temporarily use the heat strategy):

- Merge by the number of `keys` (used by Google): the statistics of Region
  with a small number of rows combined to take a weighted average
  - Pros: The distribution of the vertical axis of the heatmap is consistent
    with the distribution of Keys.
  - Cons: The hotspot information of a large number of reads and writes
    concentrated on a small number of Keys is not obviously
- Merge by the Region statistics (used by Hackathon version): cold row merge
  takes the weighted average
  - Pros: The hotspot information is obvious, we can get the details without
    zooming in.
  - Cons: unable to visually see the number of rows in the hotspot.

- Improvement 1: Combine rows with similar heat
- Improvement 2: In the TiDB mode, the boundary problem needs to be considered,
  the merged rows cannot cross-boundary
- Improvement 3: When the number of rows in the heatmap is small, the height of
  each row of the front end can be weighted according to the number of keys.

Performance issues:

The conversion of the matrix takes some time, especially the decoding and
encoding process of the key
https://github.com/pingcap/pd/issues/1837

##### Memory Optimization

According to current TiDB encoding:

```text
------------------------------------------------
t TableID _r RowID
t TableID _i IndexID IndexValues
------------------------------------------------
```

The length of the key is mostly between 20 and 40 bytes. The value of each bucket
is based on the existing 4 latitudes,  32 bytes are required. If the amount is
200W per minute. Consumption, the amount of memory required is about 99MB ~ 137MB.

In order to reduce memory overhead:

- Key storage optimization: key constant pool:

  - https://github.com/chriso/go-intern

  - https://godoc.org/github.com/chriso/go-intern

- cold message pre-delete: If the statistics of the region are small before
  adding data, it will not be collected, and it will be regarded as 0 when
  it is involved in the calculation.

- 0 value null optimization

##### Persist Statistics

If all the statistics are stored in memory, as the number of regions increases
and the time goes by, memory resources are bound to be insufficient.
In addition, we also hope that the PD restart will not lose the existingstatistics.
Therefore, consider persisting the data to the LevelDB of the current PD
(subsequently consider saving to TiKV):

Storage format:

- key: Timestamp_StartKey
- value: TimeDelta_EndKey_Stat

where:

- `Timestamp`: Time of data reporting
- `TimeDelta`: The statistic granularity. Initialized by the reporting interval
  of `TiKV`, and multiplied by the compression parameter.
- `StartKey`: The start key of the Key Range corresponding to the statistics `Bucket`
- `EndKey`: The end key of the Key Range corresponding to the statistics `Bucket`.
- `Stat`: Statistics information

#### Heatmap Data Generation

##### Generate on request (current implementation)

scans the data source according to the parameters of the front-end request and
 generates the corresponding heat map. For a description of the parameters,
 see the PD Front End Implementation chapter. The related steps are as follows:

- Determine the number of columns in the time-axis and [StartTime, EndTime)
  for each key-axis based on the requested starttime and endtime.
- Scan data source, each line of data is converted to Line, added to the
  corresponding KeyAxis according to the TS in the key.
  - WIP: Each time a KeyAxis is generated, a pre-compaction is performed first,
    to reduce the number of lines to less than 10,000.
- Get a Plane
- Perform statistical analysis of Plane to get a reasonable vertical key-axis
  division strategy.
- According to the strategy, Plane is pixelated into a numerical matrix.
- Add label information to the vertical axis of the matrix.

##### Generate at regular intervals. (Bigable implementation)

Generate a heatmap report at regular intervals. The report will be saved for
a longer period of time and can be viewed at any time.
Among them, the Daily Report of the previous week is generated every day, and
the Hourly Report of the first 4 hours is generated every hour.

Pros:

- Historical statistics of calculations that are no longer involved can be
  deleted, saving storage resources.
- After the report is generated, it can be viewed repeatedly, no need to
  calculate again, saving computing resources.

Cons:

- The report directly determines the accuracy of the data. It cannot be scaled
  up to larger precision observation data.
- The time range for each heatmap is fixed. If the heatmap is not generated, we
  must wait and cannot view it in real-time.
- The time range for each heatmap is fixed. It is not possible to analyze periodic
  hotspots over a large time range.

#### Label Decoding

Label decoding is used to display label axis and to prevent the compression
from merging across logical ranges (the border between two tables, ect).

Two modes are supported by default:

- `TiDB` mode
  - Retrieve and cache full meta information from TiDB in regular intervals
  - Split on table boundaries
  - A label may contain all information of the tables under it
- Pure `TiKV` mode
  - Split by custom rules.
  - Split by common patterns:
  - `/{str1}/{str2}/{str3}` different str1 are considered different logical ranges.
  - `/T_id/r_xxx` -> `Label {Table_id, Row_xxx}`
