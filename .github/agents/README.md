# Agents

Custom agent mode definitions. Each `.agent.md` file creates a specialized AI persona with constrained capabilities and specific behavioral rules.

| File                                               | Purpose                                                                                                                            |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| [`code-reviewer.agent.md`](code-reviewer.agent.md) | Structured, opinionated code review across any language. Covers correctness, security (OWASP), tests, maintainability, and design. |

| [`bigquery-analyst.agent.md`](bigquery-analyst.agent.md) | Read-only BigQuery analyst. Translates natural language into cost-estimated Standard SQL, dry-runs before billing, requires confirmation above 1 GB, and explains results. Never executes DML or DDL. |
See [`AGENTS.md`](../../AGENTS.md) for naming conventions and contribution guidelines.
