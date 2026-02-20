---
name: global-standards
applyTo: "**/*"
description: This file defines global development standards that apply to all code in the project, regardless of language or framework. These standards cover naming conventions, function design, security practices, error handling, testing, version control, and documentation. Adhering to these guidelines ensures that the codebase remains consistent, maintainable, secure, and of high quality across all components and services.
---

# Global Development Standards

These rules apply to all code regardless of language, framework, or project type. They represent the minimum baseline for production-grade software.

## Naming

- Names must communicate intent without a comment. If a name needs explanation, rename it.
- Avoid abbreviations unless universally understood in context (`id`, `url`, `db`, `http` are fine; `tmp`, `mgr`, `util` are not).
- Booleans and functions returning booleans: start with a verb — `isValid`, `hasPermission`, `canRetry`, `shouldRender`.
- Functions: name for what they return or accomplish, not how — `getUserById` not `dbLookupUser`.
- Configuration constants and limits: `SCREAMING_SNAKE_CASE` in every language.

## Functions and Methods

- A function must do one thing. If the name contains "and" or "or", it is doing too many things — split it.
- Hard limit: 40 lines per function. Extract named helpers when approaching this limit.
- Maximum nesting depth: 3 levels. Use early returns (guard clauses) to flatten conditionals.
- A function that has a side effect (I/O, mutation, network, time) must make that obvious — either in its name or its signature.
- Prefer pure functions: same input → same output, no hidden state changes.

## Security

- Never hardcode secrets, API keys, tokens, or passwords in source code. Use environment variables or a secrets manager.
- Validate and sanitize all external input at every system boundary (user input, API responses, file contents, environment variables, CLI args). Trust nothing from outside the process.
- Use parameterized queries for all database operations. String interpolation into queries is never acceptable under any circumstances.
- Apply least-privilege: a component must request only the permissions and data access it actually needs.
- Pin dependencies to exact versions in production lockfiles. Never use `latest`, `*`, or open version ranges in production.
- On authentication and authorization paths, fail closed (deny by default). Never fail open.
- Never expose stack traces, internal file paths, version strings, or system details in error messages shown to end users.

## Error Handling

- Never swallow exceptions silently. At minimum, log the error with enough context (operation, inputs, component) to diagnose it.
- Distinguish recoverable errors (retry / fallback possible) from unrecoverable ones (fail fast with a clear message).
- Use the most specific error/exception type available. Catch the base `Exception` or `Error` only if you will re-throw or wrap it.
- Errors that cross a service or API boundary must be mapped to a stable, documented error contract — never leak internal error types.

## Testing

- Every public function must have at minimum: one test for the happy path and one test for a failure or edge case.
- Tests must be deterministic and repeatable. No `sleep`, no random data without a fixed seed, no real network calls without a stub.
- Test names describe the scenario: `test_login_fails_when_password_is_wrong`, not `test_login_2`.
- Test behavior through public interfaces, not internal implementation details.
- A test that always passes is worse than no test. Every assertion must be capable of failing on a real bug.
- Test files live adjacent to the code they test unless the project convention specifies otherwise.

## Version Control

- Commit messages follow Conventional Commits: `<type>(<scope>): <description>`.
- Write in the imperative mood: "add feature" not "added feature" or "adds feature".
- One logical change per commit. Unrelated fixes must not be bundled together.
- Never commit commented-out code. Delete it — git history preserves it.
- Never commit `.env` files, private keys, credentials, or generated build artifacts.

## Documentation

- Document **why**, not **what**. Code shows what; comments explain non-obvious decisions, constraints, and trade-offs.
- Public APIs — exported functions, classes, HTTP endpoints — must have a doc comment describing: what it does, its parameters, its return value, and any errors it can raise.
- Keep documentation adjacent to the code it describes. Documentation that drifts out of sync is worse than no documentation.
- Do not add docstrings or comments to code you did not change unless they are factually incorrect.
