# Instructions

Always-on rules injected into every Copilot interaction for files matching the `applyTo` pattern in each file's frontmatter.

| File                                                                   | Scope              | Purpose                                                                                                          |
| ---------------------------------------------------------------------- | ------------------ | ---------------------------------------------------------------------------------------------------------------- |
| [`global-standards.instructions.md`](global-standards.instructions.md) | `**/*` (all files) | Universal coding baseline: naming, functions, security, error handling, testing, version control, documentation. |

| [`bigquery-sql.instructions.md`](bigquery-sql.instructions.md) | `**/*.sql` | Always-on rules for BigQuery SQL: no `SELECT *`, partition filter required, parameterized queries, fully-qualified references, cost documentation, DML/DDL approval gate, and PII controls. |
See [`AGENTS.md`](../../AGENTS.md) for naming conventions and contribution guidelines.
