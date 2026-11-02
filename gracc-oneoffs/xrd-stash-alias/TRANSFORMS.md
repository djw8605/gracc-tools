# OpenSearch Transform Jobs for OSDF Summary

This directory contains scripts and configurations for creating OpenSearch transform jobs that aggregate OSDF data into one summary index per year.

## Overview

The transform jobs replace the previous rollup approach with a more flexible transform-based solution.

- **Completed months** each get a one-shot transform (`osdf-summary-<year>-<MM>`).
- **The rest of the current year, from the day the installer runs (UTC),** gets one continuous "live" transform, `osdf-summary-<year>-from-<MM>-<DD>`, named for the day it starts on.
- **The days of the current month before that** get a one-shot of their own (`osdf-summary-<year>-<MM>`), which ends where the live transform starts.
- **A future year** gets one live transform from Jan 1 (`osdf-summary-<year>-from-01-01`), so next year's can be installed ahead of time (see [Installing next year ahead of time](#installing-next-year-ahead-of-time)).
- Once a live transform exists, the days from its start on belong to it, and the installer won't create one-shots for them.

Every transform:

- Processes data from `xrd-stash*` indices
- Filters data to its month (or, for a live transform, its start day through year end) using timestamp ranges
- Leaves out `/replicate:` pseudo-paths (see [Excluded data](#excluded-data))
- Groups data by day and multiple dimension fields
- Calculates min, max, sum, avg, and count metrics for filesize, read, readv, and write fields
- Outputs to a year-specific index: `osdf-summary-<year>`

### Why monthly

A transform pages through its buckets with a composite aggregation. **Every page re-scans every document its query matches.**

The old single-year 2025 transform used 1,000-bucket pages. Each page read all 16B documents of 2025, which took 9–15 min, and it would have needed months to finish.

With a one-month query, OpenSearch skips most of the 1,100+ `xrd-stash*` shards before searching. Combined with 10,000-bucket pages, the whole of Sep–Dec 2025 was rebuilt in under two hours (Oct 2026).

### Why the live transform starts on install day

A continuous transform is far slower per summary row than a one-shot (from the OpenSearch 2.11 transform source):

- **Small pages.** It writes at most 100 buckets per page (its `page_size`), against up to 10,000 for a one-shot. See [Page Size](#page-size) for why.
- **Same cost per page.** A page still takes about as long as a one-shot page, about 2 minutes on the test cluster.
- **One shard at a time.** Its first pass walks the source shard by shard, through all 1,100+ `xrd-stash*` shards.
- **No partial progress.** It saves its progress only when a run finishes, so a failure part-way through the first pass starts it over.

The first live transform, `osdf-summary-2026-from-10`, was created on Oct 27 to cover Oct 1 on, so its first pass had four weeks of data. It wrote about 3,200 summary rows an hour, so its first pass would have taken days, and an HDD-node outage restarted it once. On Oct 29 it was replaced by a one-shot for Oct 1–28 and a live transform from Oct 29.

So the live transform starts on the day the installer runs, and its first pass has almost nothing to summarize. After that, each hourly run only revisits the shards that received new data.

### Late data in the current month's one-shot

The one-shot for the live transform's first month ends at midnight UTC on the install day, and it runs once. Records for those days that arrive after it has run are not summarized. To pick them up, re-create that one-shot a day or two later:

```bash
python install_transforms.py \
  --url https://your-opensearch-host:9200 \
  --user admin \
  --start-year 2026 --end-year 2026 \
  --start-month 10 --end-month 10 \
  --force
```

With a live transform in place, that month's only job is the one-shot, and the live transform is left alone. The re-created one-shot has the same ID, so it overwrites its own documents and `--clean-target` isn't needed.

## Files

- `example-transform.json` - Example transform configuration (older yearly format, for reference)
- `install_transforms.py` - Python script to install the transform jobs
- `install_rollup.py` - Legacy script for installing rollup jobs (for reference)
- `example-rollup.json` - Legacy rollup configuration (for reference)

## Installation

### Prerequisites

```bash
pip install -r requirements.txt
```

### Install All Transform Jobs (2019-2025)

```bash
export OPENSEARCH_PASSWORD=your-password
python install_transforms.py \
  --url https://your-opensearch-host:9200 \
  --user admin
```

### Options

```
--url, -u          Base URL of OpenSearch (required)
--user             Username for basic auth (required)
--password         Password for basic auth (default: $OPENSEARCH_PASSWORD)
--start-year       Start year (default: 2019)
--end-year         End year (default: 2025)
--start-month      First month to install in each year (default: 1)
--end-month        Last month to install in each year (default: 12)
--insecure         Skip TLS certificate verification
--no-start         Create the transforms disabled, without starting them
--force            Delete and recreate transforms that already exist
--clean-target     Delete the existing summary documents in each transform's date range before creating it
--dry-run          Only print which transforms would be installed; change nothing
```

When a year is installed as monthly transforms, the installer stops and deletes that year's old single-year transform (`osdf-summary-<year>`), if one exists. Its documents stay in the target index.

With `--no-start`, the transforms are created disabled. Start them yourself with `_start`, a few at a time if the cluster is busy. Each running transform is a full-month scan per page.

### Install a Subset of Years or Months

```bash
python install_transforms.py \
  --url https://your-opensearch-host:9200 \
  --user admin \
  --start-year 2025 \
  --end-year 2025 \
  --start-month 9 \
  --end-month 12
```

### Rebuild Months That Were Already Summarized

Target document IDs are derived from the transform ID. A month first summarized by a different transform, such as the old yearly one, therefore keeps its old documents next to the new ones, and dashboards double count it. Pass `--clean-target` to delete that month's documents from the target index before its transform is created:

```bash
python install_transforms.py \
  --url https://your-opensearch-host:9200 \
  --user admin \
  --start-year 2025 --end-year 2025 \
  --start-month 9 --end-month 12 \
  --clean-target
```

Add `--force` as well if the monthly transforms themselves already exist and should be recreated.

`--clean-target` empties the whole range before the transform starts, so queries return far less data until it finishes. When the months were summarized by the **same transform IDs** (for example, after changing the exclusions), rebuild them in place instead:

1. Record the target index's highest `_seq_no`. Every `osdf-summary-<year>` index has one primary shard, so one number covers it:

   ```bash
   curl -s -u admin:password -H 'Content-Type: application/json' \
     "https://your-opensearch-host:9200/osdf-summary-2026/_search" \
     -d '{"size": 0, "aggs": {"m": {"max": {"field": "_seq_no"}}}}'
   ```

2. Recreate and start the transform with `--force`, without `--clean-target`. It writes the same document IDs, so each row is overwritten when the transform reaches its bucket, and the old rows stay readable until then.
3. When it has finished, delete the rows in its range that it did not rewrite: buckets that no longer exist, such as ones made up only of excluded records.

   ```bash
   curl -s -u admin:password -H 'Content-Type: application/json' \
     "https://your-opensearch-host:9200/osdf-summary-2026/_delete_by_query" \
     -d '{"query": {"bool": {"filter": [
           {"range": {"@timestamp": {"gte": "2026-07-01T00:00:00Z", "lt": "2026-08-01T00:00:00Z"}}},
           {"range": {"_seq_no": {"lte": SEQ_NO_FROM_STEP_1}}}]}}}'
   ```

While a month is being rebuilt, its totals mix rewritten rows and old ones, but nothing goes missing. Keep the date range in step 3: the other months' rows have older sequence numbers too.

The Oct 30 – Nov 2, 2026 rebuild ran this way, through a driver script that also checked each month against the source.

### Installing Next Year Ahead of Time

Run the installer for next year before it starts, and it creates one live transform from Jan 1 instead of monthly one-shots:

```bash
python install_transforms.py \
  --url https://your-opensearch-host:9200 \
  --user admin \
  --start-year 2027 --end-year 2027
```

This avoids a manual step in January and leaves no gap at the year boundary. Until Jan 1, the transform has nothing in range:

- Its first pass still visits every `xrd-stash*` shard once (the Oct 29, 2026 live transform's took about 2.5 hours, most of it on shards with nothing in range).
- Each hourly run then checks only the shards that received new data, and finds nothing.
- Its target index (`osdf-summary-<year>`) exists, empty, from install on.

`osdf-summary-2027-from-01-01` was installed this way on Oct 30, 2026.

### Skip TLS Verification (for development)

```bash
python install_transforms.py \
  --url https://localhost:9200 \
  --user admin \
  --insecure
```

## Transform Configuration

Each transform job includes:

### Grouping Dimensions
- `@timestamp` - Date histogram with 1-day calendar interval (UTC)
- `dirname1` - First directory component
- `dirname2` - Second directory component
- `logical_dirname` - Logical directory name
- `server` - Server identifier
- `server_hostname` - Server hostname
- `site` - Site identifier
- `user_domain` - User domain

### Metrics
For each of the following fields: `filesize`, `read`, `readv`, `write`
- `min` - Minimum value
- `max` - Maximum value
- `sum` - Total sum
- `avg` - Average value
- `count` - Value count
- `doc_count` - Document count per bucket (value_count on `@timestamp`)

### Page Size
One-shot transforms use `page_size` 10000, the largest a transform accepts. Fewer pages means fewer full scans of the source.

The live transform uses `page_size` 100. A continuous transform re-queries each page of changed buckets with one clause per group for every bucket, and adds the `data_selection_query` on top. All of that has to fit in Lucene's limit of 1024 clauses, and Lucene counts a range query as 2:

- OpenSearch sizes the page at min(`page_size`, 1024 / (groups + 1)), which is 113 buckets with our 8 groups. Each bucket costs 9 clauses (the day range is 2), so 113 buckets take 1017.
- The `data_selection_query` costs 15: the `@timestamp` range (2), the `/replicate:` prefix (1), and the six garbage-counter ranges (2 each). 1017 + 15 is over the limit, and the transform fails its first search that has data with `too_many_nested_clauses` ("Failed to search data in source indices").
- With `page_size` 100, the buckets take 900, which leaves room. Before the garbage-counter exclusion, the query cost only 3, so `page_size` 1000 (113 in practice) worked.

After its first pass, the live transform only recomputes the buckets that new data touches.

### Excluded Data
Documents whose `dirname1` starts with `/replicate:` are left out through the transform's `data_selection_query`.

These are Purdue EOS replication requests (site `T2_US_Purdue_EOS`, Sep 2025 – Jan 2026: about 13M documents in 2025 and 2.2M in 2026). Each one has a unique `dirname1`/`dirname2` like `/replicate:0000ca0f`, so each becomes its own bucket. On 2025-09-12 alone that was 1.6M buckets, against a few thousand on a normal day. The prefixes are listed in `EXCLUDED_DIRNAME1_PREFIXES` in `install_transforms.py`.

Documents with a garbage transfer counter are left out too: any of `read`, `readv` or `write` negative or over 1 PB (`TRANSFER_FIELDS` and `MAX_TRANSFER_BYTES` in `install_transforms.py`).

- There were 323,597 of them in `xrd-stash*`, from Dec 2025 to Oct 2026, and they were still arriving.
- 99.5% are close records with no matching open (`dirname1` is "unknown directory"), and most come from servers reporting as 127.0.0.1. The worst site was FDP_OSDF_ORIGIN.
- A single one adds up to 9.2 EB to a day's summed bytes, and the negative ones subtract.
- They are dropped entirely, including from the record counts, because they don't describe real transfers.
- The raw documents are unchanged.

The months they affected (2025-12 and 2026-01 to 2026-10) were rebuilt with this exclusion from Oct 30 to Nov 2, 2026. Every month's summed `transform._doc_count` matched the source count afterward.

### Scheduling
- The **live transform** (`osdf-summary-<year>-from-<MM>-<DD>`) runs in **continuous mode** every 1 hour, processing new data as it arrives
- **Monthly** transforms run as **one-shot** jobs that process their month once and then stop (status `finished`, job disabled)
- All transforms process historical data when first created
- **After the year ends**, the live transform still owns its days, and keeps running hourly with nothing new in range. Next year's live transform covers the new year, whether it was installed ahead of time or is installed after Jan 1. To turn the old one's days into one-shots:
  1. Stop and delete the old live transform.
  2. Re-run the installer for that year with `--clean-target --force`. `--force` replaces the partial one-shot of the month the live transform started in with a full-month one.

## Differences from Rollup

1. **Year-based Target Indices**: Each year has its own target index, filled by that year's monthly (or continuous) transforms
2. **Date Range Filtering**: Each transform only processes data from its specific month or year
3. **Calendar Interval**: Buckets are calendar days in UTC
4. **Explicit Aggregations**: Each metric is explicitly named (e.g., `filesize_min`, `filesize_max`)

## Managing Transform Jobs

### Check Transform Status

```bash
curl -X GET "https://your-opensearch-host:9200/_plugins/_transform/osdf-summary-2025-09/_explain" \
  -u admin:password
```

`transform_metadata.status` is `init`, `started`, `finished` or `failed`. A job that is disabled (`enabled: false` on `GET _plugins/_transform/<id>`) but whose status is still `started` is stuck: it will not run again and does not show as failed. This is how the old yearly 2025 transform sat idle from May to Oct 2026. Start it again with `_start`.

### Stop a Transform

```bash
curl -X POST "https://your-opensearch-host:9200/_plugins/_transform/osdf-summary-2025-09/_stop" \
  -u admin:password
```

### Start a Transform

```bash
curl -X POST "https://your-opensearch-host:9200/_plugins/_transform/osdf-summary-2025-09/_start" \
  -u admin:password
```

### Delete a Transform

```bash
curl -X DELETE "https://your-opensearch-host:9200/_plugins/_transform/osdf-summary-2025-09" \
  -u admin:password
```

## Troubleshooting

### View Transform Logs

Check OpenSearch logs for transform-related errors:

```bash
# On the OpenSearch server
tail -f /var/log/opensearch/opensearch.log | grep transform
```

### Check Transform Statistics

```bash
curl -X GET "https://your-opensearch-host:9200/_plugins/_transform/osdf-summary-2025-09/_explain" \
  -u admin:password
```

### Verify Target Index

```bash
curl -X GET "https://your-opensearch-host:9200/osdf-summary-2025/_count" \
  -u admin:password
```

To check a month is complete, compare the sum of `transform._doc_count` in the target for that month with a `_count` of the source using the same query as the transform. They should be equal.

## Notes

- The live transform runs continuously and processes new data every hour
- Historical data is processed when the transform is first created
- Each transform only processes data from its designated month (or, for the live transform, from its start day to year end)
- Target indices are automatically created if they don't exist
