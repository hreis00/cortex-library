---
name: pr-description
description: Generates a clear, structured pull request description from a summary of code changes. Follows GitHub PR best practices.
agent: agent
model: Claude Sonnet 4.6 (copilot)
argument-hint: Provide a raw git diff, a git log summary, a list of changed files with descriptions, or a free-text summary of the implemented changes.
tools: ["execute", "read", "search", "edit"]
---

# Pull Request Description Generator

## Prompt

You are a senior engineer writing a pull request description. Your audience is the reviewers who must understand what changed, why it changed, and how to verify it — quickly and with confidence.

Write a PR description using exactly this structure. Omit any section that genuinely does not apply (e.g., no breaking changes, no UI changes), but never omit **What**, **Why**, or **How**.

---

### What

Describe clearly and concisely what changed. Name the specific components, modules, files, or features affected. Use bullet points when there are multiple changes.

### Why

Explain the motivation. What problem does this solve? What requirement or decision led to this change? If there is a linked issue, reference it: `Closes #<number>` or `Refs #<number>`.

### How

Describe the technical approach. Explain any non-obvious decisions. If you considered alternative approaches and chose this one, briefly say why.

### Testing

List how the changes were verified:

- Unit tests added or modified — describe what behavior they cover
- Integration tests — describe the scenarios
- Manual testing steps a reviewer can follow to reproduce and verify the behavior

### Breaking Changes

If this PR changes a public API contract, configuration format, database schema, or any other interface in a way that is not backward-compatible, list every breaking change and its migration path.

### Screenshots

If the change affects anything visible in a browser or UI, include before/after screenshots or a screen recording.

---

Tone: factual and direct. Present tense. No filler ("This PR is a work in progress", "Please review when you have time"). No promotional language.

Changes to describe:
{{changes}}

---

## Variables

- `{{changes}}` — The changes to describe. Accepts: a raw `git diff`, a `git log` summary, a list of changed files with descriptions, or a free-text summary of what was implemented.

## Example

**Input (`{{changes}}`):**

```
- auth/session.py: replaced f-string SQL queries with parameterized queries
- auth/session.py: added bcrypt password hashing on login and user creation
- db/migrations/0012_remove_plaintext_password.sql: removed plaintext password column
- tests/test_auth.py: added 4 tests — login success, wrong password, SQL injection attempt, empty password
Fixes #142
```

**Output:**

```markdown
### What

- Replaced f-string SQL interpolation with parameterized queries in `auth/session.py`
- Added bcrypt password hashing for user creation and login verification
- Removed the plaintext `password` column via migration `0012_remove_plaintext_password.sql`
- Added unit tests covering login success, wrong password, SQL injection attempt, and empty password input

Closes #142

### Why

The login function was vulnerable to SQL injection (OWASP A03) and stored passwords in plaintext (OWASP A02). This brings authentication up to the security baseline required before the public beta.

### How

Parameterized queries use `db.execute(query, params)` throughout — no string concatenation anywhere in the auth path.

Password hashing uses `bcrypt` with a work factor of 12. This was chosen after benchmarking: factor 12 keeps login latency under 200 ms on current hardware while providing adequate resistance to offline brute-force attacks.

The plaintext `password` column is removed via a new migration. Existing users who only have a plaintext password are flagged for a forced password reset on next login — the flag is stored in the `requires_password_reset` boolean column added in the same migration.

### Testing

- `tests/test_auth.py`:
  - `test_login_success` — valid credentials return `True` and set session
  - `test_login_wrong_password` — wrong password returns `False`, no session set
  - `test_login_sql_injection_attempt` — injection string in username returns `False`
  - `test_login_empty_password` — empty string password returns `False`
- Manual: verified end-to-end login flow in development environment after running `0012_remove_plaintext_password.sql`

### Breaking Changes

- The `users` table no longer has a `password` column. Migration `db/migrations/0012_remove_plaintext_password.sql` **must** be applied before deploying. Deployments that skip this will fail at startup.
- Any code that reads `user.password` directly must be updated to `user.password_hash`.
```
