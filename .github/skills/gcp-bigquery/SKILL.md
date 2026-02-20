---
name: gcp-bigquery
description: BigQuery data platform knowledge covering standard SQL, partitioning, clustering, slot-based pricing, INFORMATION_SCHEMA, query optimization, schema design, and production security practices. Use when designing queries, estimating costs, reviewing schemas, or debugging job performance.
argument-hint: "Ask about partitioning strategy, clustering keys, cost estimation, INFORMATION_SCHEMA views, bq CLI usage, schema design, or security controls."
---

# BigQuery

BigQuery is Google Cloud's fully-managed, serverless data warehouse. It separates compute (slots) from storage, enabling elastic scaling and on-demand or reserved capacity pricing. Understanding how BigQuery processes queries — especially how it scans bytes — is essential for both cost control and performance.

## Core Concepts

### Standard SQL vs. Legacy SQL

BigQuery supports two SQL dialects. **Standard SQL** (default) is ANSI-compliant and the only dialect that supports all modern BigQuery features. Legacy SQL is deprecated — never use it.

| Feature | Standard SQL | Legacy SQL |
|---|---|---|
| Nested/repeated fields | Native `STRUCT`, `ARRAY` | `RECORD`, `REPEATED` — limited ops |
| DML | `INSERT`, `UPDATE`, `DELETE`, `MERGE` | Not supported |
| Scripting | `IF`, `LOOP`, variables | Not supported |
| Status | Current | Deprecated |

### Storage Model

- **Dataset**: container for tables, views, and routines. Scoped to a GCP project.
- **Table**: structured data in columnar storage (Capacitor format).
- **View**: logical view; no storage cost; billed for underlying scan on each query.
- **Materialized view**: pre-computed results refreshed incrementally; reduces query cost.
- **External table**: data stays in GCS, Bigtable, or Drive; BigQuery queries it in place.

### Partitioning

Partitioning divides a table into segments. BigQuery only scans partitions matching the filter — this is the single most effective cost-reduction mechanism.

| Type | Column type | Use case |
|---|---|---|
| Ingestion-time (`_PARTITIONTIME`) | Automatic timestamp | Append-heavy event tables |
| Date/timestamp column | `DATE`, `TIMESTAMP`, `DATETIME` | Business-date tables |
| Integer range | `INT64` | Numeric ID-range sharding |

**Partition pruning requires a direct filter** on the partition column. A query without one performs a full table scan regardless of data volume.

```sql
-- Full scan — no partition filter
SELECT user_id, event_type FROM `project.dataset.events`;

-- Partition-pruned — scans only matching partition(s)
SELECT user_id, event_type
FROM `project.dataset.events`
WHERE DATE(event_timestamp) >= '2026-01-01';
```

A partitioned table can hold up to 10,000 partitions. Use expiration policies (`partition_expiration_days`) to prevent partition sprawl.

### Clustering

Clustering co-locates rows with similar values in adjacent storage blocks. BigQuery uses block-level metadata to skip non-matching blocks (block pruning).

- Applied after partitioning; specify up to **4 columns**, ordered by selectivity (most selective first).
- Effective for high-cardinality columns frequently used in `WHERE`, `JOIN`, `GROUP BY`, or `ORDER BY`.
- Unlike partitioning, clustering is a best-effort optimization — it does not guarantee a fixed cost reduction.

```sql
CREATE TABLE `project.dataset.events`
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id, event_type
AS SELECT ...;
```

### Slot-Based Pricing

A **slot** is a unit of BigQuery compute (CPU + memory). Every query consumes slots during execution.

| Model | Description | Best for |
|---|---|---|
| On-demand | Charged per byte scanned (~$6.25/TB) | Sporadic, unpredictable workloads |
| Autoscale (editions) | Slots scale to demand; billed per slot-hour | Variable workloads needing predictability |
| Flat-rate / capacity commitments | Reserved slots; no per-byte charge | Steady, high-volume workloads |

Under on-demand pricing, **bytes scanned drives cost** — not rows returned, not query complexity. Partition pruning and column selection are the primary cost levers.

### INFORMATION_SCHEMA

`INFORMATION_SCHEMA` is a set of read-only views exposing metadata about datasets, tables, columns, jobs, and more. No additional permissions are needed beyond existing data access.

| View | Scope | Contains |
|---|---|---|
| `INFORMATION_SCHEMA.TABLES` | Dataset | Table names, row count, size, type |
| `INFORMATION_SCHEMA.COLUMNS` | Dataset | Column names, types, modes, descriptions |
| `INFORMATION_SCHEMA.PARTITIONS` | Dataset | Per-partition row counts and sizes |
| `INFORMATION_SCHEMA.JOBS_BY_PROJECT` | Project | All job history: bytes processed, state, duration, query text |
| `INFORMATION_SCHEMA.JOBS_BY_USER` | Project | Same, filtered to the current user |
| `INFORMATION_SCHEMA.TABLE_STORAGE` | Project | Logical vs physical storage bytes |

```sql
-- Inspect table schema
SELECT column_name, data_type, is_nullable
FROM `project.dataset.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'events'
ORDER BY ordinal_position;

-- Last 24h job history — bytes processed per user
SELECT
  user_email,
  SUM(total_bytes_processed) AS bytes_processed
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR)
  AND job_type = 'QUERY'
  AND state = 'DONE'
GROUP BY user_email
ORDER BY bytes_processed DESC;
```

`INFORMATION_SCHEMA.JOBS_BY_PROJECT` requires `bigquery.jobs.listAll` (typically `roles/bigquery.resourceViewer` or higher).

---

## Best Practices

### Cost Controls

1. **Always dry-run before executing.** The `bq` CLI `--dry_run` flag returns `totalBytesProcessed` without billing. Use it for every non-trivial query.
   ```bash
   bq query --use_legacy_sql=false --dry_run \
     'SELECT event_id FROM `project.dataset.events` WHERE DATE(ts) = "2026-01-15"'
   ```

2. **Set a byte-billing cap.** Use `--maximum_bytes_billed` to prevent accidental full-table scans from billing unexpectedly.
   ```bash
   bq query --use_legacy_sql=false --maximum_bytes_billed=1073741824 ...  # 1 GB hard cap
   ```

3. **Select only needed columns.** BigQuery is columnar — every unused column scanned costs the same as a used one. Never `SELECT *` in production.

4. **Prefer materialized views for repeated aggregations.** A materialized view is refreshed incrementally; querying it scans only the pre-computed result.

5. **Set table and partition expiration policies** on ephemeral or staging tables to prevent unbounded storage growth.
   ```sql
   ALTER TABLE `project.dataset.staging_events`
   SET OPTIONS (expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY));
   ```

### Query Optimization

1. **Filter early; filter on partition columns first.** Partition pruning only applies when the filter is a direct comparison on the partition column — before any function is applied.

2. **Never wrap a partition column in a function on the filter left-hand side.** This breaks partition pruning.
   ```sql
   -- BAD: disables partition pruning
   WHERE CAST(event_timestamp AS DATE) = '2026-01-01'

   -- GOOD: direct comparison preserves partition pruning
   WHERE event_timestamp >= '2026-01-01' AND event_timestamp < '2026-01-02'
   ```

3. **Use `APPROX_COUNT_DISTINCT` for cardinality estimates** when exact counts are not required; it uses far fewer resources.

4. **Use window functions instead of self-joins.**
   ```sql
   -- BAD: self-join to compute running total
   SELECT a.id, SUM(b.amount) FROM orders a JOIN orders b ON b.id <= a.id GROUP BY a.id;

   -- GOOD: window function
   SELECT id, SUM(amount) OVER (ORDER BY id) FROM orders;
   ```

5. **CTEs are inlined, not cached.** BigQuery does not materialize CTEs unless explicitly referenced via `TEMP TABLE`. Do not assume a CTE is cached when referenced multiple times.

6. **Avoid `ORDER BY` without `LIMIT` on large result sets.** Sorting writes the full result to a shuffle stage, which is often the most expensive step in a query.

### Schema Design

1. **Prefer nested and repeated fields over normalized joins.** BigQuery is optimized for denormalized, wide tables. Multi-table joins create shuffle stages that dominate cost and latency.

2. **Use `STRUCT` for logically grouped fields** that are always accessed together.
   ```sql
   CREATE TABLE `project.dataset.events` (
     event_id        STRING    NOT NULL,
     event_timestamp TIMESTAMP NOT NULL,
     user            STRUCT<id STRING, email STRING, country STRING>,
     properties      JSON
   )
   PARTITION BY DATE(event_timestamp)
   CLUSTER BY user.country;
   ```

3. **Document columns with descriptions.**
   ```sql
   ALTER TABLE `project.dataset.events`
   ALTER COLUMN user SET OPTIONS (
     description = 'Nested user record. user.id is the stable UUID; user.email is PII.'
   );
   ```

### Security

1. **Grant access at the dataset level** unless table-level isolation is a hard requirement. Dataset-level IAM is simpler to audit and less prone to permission drift.

2. **Use Policy Tags for PII/PHI columns.** Column-level security via Data Catalog Policy Tags prevents unauthorized reads without query-layer workarounds.

3. **Use row-level security for multi-tenant tables** instead of maintaining separate tables per tenant.
   ```sql
   CREATE ROW ACCESS POLICY region_filter
   ON `project.dataset.events`
   GRANT TO ('group:eu-analysts@example.com')
   FILTER USING (region = 'EU');
   ```

4. **Use service accounts with minimum required roles.** For read-only workloads: `roles/bigquery.dataViewer` + `roles/bigquery.jobUser`. Never grant `roles/bigquery.admin` to application service accounts.

5. **Prefer authorized views over data copies** when sharing a table subset across projects. An authorized view enforces access without duplicating data.

6. **Never hardcode credentials or project IDs** in query text or scripts. Use Application Default Credentials (ADC) and environment-scoped configuration.

---

## Patterns

### Inspect a Table Before Querying

```bash
# Show schema
bq show --schema --format=prettyjson project:dataset.tablename

# Preview first 10 rows — no billing
bq head -n 10 project:dataset.tablename
```

### Dry-Run Cost Estimate

```bash
bq query \
  --use_legacy_sql=false \
  --dry_run \
  --project_id=my-project \
  'SELECT user_id, COUNT(*) AS events
   FROM `my-project.analytics.events`
   WHERE DATE(event_timestamp) BETWEEN "2026-01-01" AND "2026-01-31"
   GROUP BY user_id'
# Output: Query successfully validated. Running this query will process 1234567890 bytes.
```

### Parameterized Query via bq CLI

```bash
bq query \
  --use_legacy_sql=false \
  --parameter='start_date:DATE:2026-01-01' \
  --parameter='end_date:DATE:2026-01-31' \
  --parameter='user_id:STRING:usr_abc123' \
  'SELECT event_type, COUNT(*) AS n
   FROM `my-project.analytics.events`
   WHERE DATE(event_timestamp) BETWEEN @start_date AND @end_date
     AND user_id = @user_id
   GROUP BY event_type'
```

### Fetch Job Execution Metadata

```sql
SELECT
  job_id,
  query,
  total_bytes_processed,
  total_slot_ms,
  TIMESTAMP_DIFF(end_time, start_time, SECOND) AS duration_seconds
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE job_id = 'bqjob_r1234567890_abcdef'
  AND DATE(creation_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY);
```

### Check Partition Metadata

```sql
SELECT
  partition_id,
  total_rows,
  total_logical_bytes,
  last_modified_time
FROM `project.dataset.INFORMATION_SCHEMA.PARTITIONS`
WHERE table_name = 'events'
ORDER BY partition_id DESC
LIMIT 20;
```

### Create a Deduplicated Snapshot Table

```sql
-- APPROVED: scheduled dedup job — data-eng team
CREATE OR REPLACE TABLE `project.dataset.events_deduped`
PARTITION BY DATE(event_timestamp)
CLUSTER BY user_id
AS
SELECT * EXCEPT (row_num)
FROM (
  SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY inserted_at DESC) AS row_num
  FROM `project.dataset.events`
)
WHERE row_num = 1;
```

---

## Anti-Patterns

### SELECT * in Production Queries

```sql
-- BAD: scans all columns; expensive on wide tables
SELECT * FROM `project.dataset.events`;

-- GOOD: enumerate only required columns
SELECT event_id, user_id, event_type, event_timestamp
FROM `project.dataset.events`
WHERE DATE(event_timestamp) = CURRENT_DATE();
```

### Missing Partition Filter

```sql
-- BAD: full table scan on a partitioned table
SELECT COUNT(*) FROM `project.dataset.events` WHERE user_id = 'usr_abc123';

-- GOOD: partition filter limits scan
SELECT COUNT(*)
FROM `project.dataset.events`
WHERE DATE(event_timestamp) = '2026-01-15'
  AND user_id = 'usr_abc123';
```

### Function Applied to Partition Column

```sql
-- BAD: EXTRACT() disables partition pruning
WHERE EXTRACT(YEAR FROM event_timestamp) = 2026

-- GOOD: range comparison preserves partition pruning
WHERE event_timestamp >= '2026-01-01' AND event_timestamp < '2027-01-01'
```

### String Concatenation into Queries

```python
# BAD: SQL injection risk
query = f"SELECT * FROM `{dataset}.events` WHERE user_id = '{user_input}'"

# GOOD: parameterized query
from google.cloud import bigquery
job_config = bigquery.QueryJobConfig(
    query_parameters=[
        bigquery.ScalarQueryParameter("user_id", "STRING", user_input)
    ]
)
query = "SELECT * FROM `project.dataset.events` WHERE user_id = @user_id"
client.query(query, job_config=job_config)
```

### Hardcoded Project and Dataset Names

```sql
-- BAD: breaks across environments; no single point of change
FROM analytics.events

-- GOOD: fully-qualified; substituted via environment config or templating
FROM `my-project.analytics.events`
```

### COUNT(*) Without Filters on a Large Table

```sql
-- BAD: wastes slots; use metadata instead
SELECT COUNT(*) FROM `project.dataset.events`;

-- GOOD: metadata-only count from INFORMATION_SCHEMA
SELECT total_rows
FROM `project.dataset.INFORMATION_SCHEMA.TABLES`
WHERE table_name = 'events';
```

---

## Tools & Commands

### bq CLI

```bash
# Authenticate
gcloud auth application-default login
gcloud config set project my-project

# Show table schema
bq show --schema project:dataset.table

# Preview data (no billing)
bq head -n 10 project:dataset.table

# Dry-run a query
bq query --use_legacy_sql=false --dry_run '<SQL>'

# Run a query with a 1 GB byte cap
bq query --use_legacy_sql=false --maximum_bytes_billed=1073741824 '<SQL>'

# List tables in a dataset
bq ls project:dataset

# Show dataset metadata
bq show project:dataset

# List recent jobs
bq ls --jobs --all --max_results=20 --project_id=my-project
```

### gcloud CLI

```bash
# IAM: view bindings on a dataset
bq get-iam-policy project:dataset

# IAM: grant dataViewer to a service account
bq add-iam-policy-binding \
  --member='serviceAccount:sa@project.iam.gserviceaccount.com' \
  --role='roles/bigquery.dataViewer' \
  project:dataset

# List datasets in a project
gcloud alpha bq datasets list --project=my-project
```

---

## References

- [BigQuery Standard SQL reference](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax)
- [Partitioned tables overview](https://cloud.google.com/bigquery/docs/partitioned-tables)
- [Clustering overview](https://cloud.google.com/bigquery/docs/clustered-tables)
- [BigQuery pricing](https://cloud.google.com/bigquery/pricing)
- [INFORMATION_SCHEMA views](https://cloud.google.com/bigquery/docs/information-schema-intro)
- [bq CLI reference](https://cloud.google.com/bigquery/docs/bq-command-line-tool)
- [Column-level security (Policy Tags)](https://cloud.google.com/bigquery/docs/column-level-security-intro)
- [Row-level security](https://cloud.google.com/bigquery/docs/row-level-security-intro)
- [Authorized views](https://cloud.google.com/bigquery/docs/authorized-views)
- [Query optimization best practices](https://cloud.google.com/bigquery/docs/best-practices-performance-overview)
