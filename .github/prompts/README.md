# Prompts

Parameterized slash-command prompts for repeatable, focused tasks. Invoke with `/` in Copilot Chat.

| File                           | Purpose                                                                                                                                                       |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`pr.prompt.md`](pr.prompt.md) | Full PR workflow: validates commits against Conventional Commits, pushes the branch, generates a structured title and description, and opens the PR via `gh`. |

| [`bigquery-query.prompt.md`](bigquery-query.prompt.md) | Translates a natural language question into a BigQuery SQL query, dry-runs the cost estimate, gates execution at 1 GB, then executes and explains results. |
| [`bigquery-explain.prompt.md`](bigquery-explain.prompt.md) | Fetches execution metadata for a BigQuery job or query from `INFORMATION_SCHEMA.JOBS_BY_PROJECT`, explains stage breakdown, and recommends optimizations. |
See [`AGENTS.md`](../../AGENTS.md) for naming conventions and contribution guidelines.
