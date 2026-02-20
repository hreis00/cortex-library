---
name: bigquery-sql
applyTo: "**/*.sql"
description: Always-on rules for BigQuery SQL files. Enforces cost controls, security boundaries, and query correctness required for production BigQuery environments.
---

# BigQuery SQL Standards

## Column Selection

- Never use `SELECT *` in production queries or views. Always enumerate columns explicitly.
- Exclude columns not needed by the consumer; every unnecessary column increases bytes scanned and query cost.

## Partition Filters

- Every query against a partitioned table must include a `WHERE` filter on the partition column.
- Filters on a partition column must use direct comparison or range operators (`=`, `<`, `>`, `BETWEEN`, `>=`, `<=`). Never wrap a partition column in a function (`CAST`, `EXTRACT`, `DATE()`, `FORMAT()`) on the left side of the filter — doing so disables partition pruning.
- Place the partition filter as the first condition in the `WHERE` clause.

## Table References

- Always use fully-qualified table references: `` `project.dataset.table` ``. Unqualified names resolve to the default project and are ambiguous across environments.
- Never hardcode project IDs, dataset names, or environment labels as string literals inside query bodies. Use query parameters, templating variables (`{{project}}`), or substitution tokens instead.

## Parameterization

- All user-supplied or runtime-variable values must be passed as query parameters (`@param_name` syntax). Never concatenate external values into SQL strings.
- Parameters must use the narrowest available type (`DATE`, `INT64`, `STRING`). Never use `ANY TYPE` for external inputs.

## DML and DDL

- `INSERT`, `UPDATE`, `DELETE`, and `MERGE` statements are forbidden unless the statement is preceded by the comment `-- APPROVED: <reason> | <author> | <date>`.
- DDL (`CREATE TABLE`, `DROP TABLE`, `ALTER TABLE`) follows the same rule.
- Queries that modify data must include a dry-run cost estimate comment immediately above the statement.

## Cost Documentation

- Any query that joins two or more tables, scans a table with no partition filter, or uses a subquery over an unbounded table must include a comment on the line immediately before the query:
  `-- estimated: <N> GB` (value obtained from `bq --dry_run` output).
- Queries estimated above 1 GB must not be merged or deployed without a second reviewer's approval noted in the PR.

## Identifiers

- Use backtick-quoted identifiers for all table and dataset references: `` `project.dataset.table` ``.
- Column aliases must use `snake_case`. Never use `camelCase` or quoted aliases with spaces.
- All identifiers (table names, column names, CTEs) must be in `snake_case`.

## Schema Awareness

- Before writing a query against an unfamiliar table, inspect its schema using `bq show --schema` or `INFORMATION_SCHEMA.COLUMNS`. Do not guess column names.
- Document the source table and its partition key in a leading comment on any multi-table query.

## Security

- Never select or surface columns tagged with a Policy Tag unless the accessor's role has been confirmed as holding the required taxonomy permission.
- Never include raw user identifiers, email addresses, or other PII/PHI in query results returned to shared dashboards, exports, or logs.
- Production dataset queries must include a `--maximum_bytes_billed` cap. The only exception is queries whose dry-run estimate is below 100 MB.
- Never embed credentials, service account keys, or OAuth tokens anywhere in SQL files.
