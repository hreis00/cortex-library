---
name: git-conventional-commits
description: Git workflow conventions covering commit message format, branch naming, and pull request practices. Based on the Conventional Commits specification. Applicable to any project using Git.
---

# Git: Conventional Commits & Workflow

## Overview

This skill defines the git conventions that keep a history readable, automation-friendly, and meaningful as a project scales. These conventions apply to any project using Git, regardless of language or framework.

- Structured commit messages that tools (changelogs, semantic versioning, CI) can parse
- Branch naming that communicates purpose at a glance
- Pull request practices that enable efficient review

---

## Core Concepts

### Commit Message Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

**Header rules:**

- `<type>` — lowercase; from the table below
- `<scope>` — optional; names the module, component, or area changed (e.g., `auth`, `api`, `cart`)
- `<description>` — imperative present tense; no trailing period; ≤ 72 characters total header
- If a body is included, separate it from the header with one blank line

### Commit Types

| Type       | When to use                                             |
| ---------- | ------------------------------------------------------- |
| `feat`     | New feature visible to users or API consumers           |
| `fix`      | Bug fix (not a style fix or test fix)                   |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `test`     | Adding or correcting tests only                         |
| `docs`     | Documentation changes only                              |
| `chore`    | Maintenance: dependency updates, build config, tooling  |
| `perf`     | Performance improvement without behavior change         |
| `ci`       | CI/CD pipeline configuration changes                    |
| `revert`   | Reverts a previous commit                               |

### Breaking Changes

Indicate a breaking change in two ways simultaneously:

1. Append `!` after the type/scope: `feat(api)!: rename user endpoint`
2. Add a `BREAKING CHANGE:` trailer in the commit body:

```
feat(api)!: rename /user to /users

The endpoint path changed from /user to /users for REST naming consistency.

BREAKING CHANGE: All clients must update endpoint references.
Deployment requires updating reverse-proxy routing before the new build goes live.
```

---

## Best Practices

1. **One logical change per commit.** If the message needs "and", split the commit.
2. **Commit during development; squash before merge.** Preserve a clean history on the default branch.
3. **Use scope consistently.** Once a team uses `fix(auth)`, every auth fix should use `fix(auth)`. A scope mismatch is a signal the commit is doing too much.
4. **The body explains why, not what.** The diff shows what changed. The body captures the decision — constraints, trade-offs, rejected alternatives.
5. **Reference issues in the footer**, not the description: `Closes #42` or `Refs #17`.
6. **Never amend or force-push commits that others have pulled.** Rewrite history only on your own branches before a PR is opened.

---

## Branch Naming

Pattern: `<type>-<short-description>` — matching the same type vocabulary as commits.

```
feat-user-authentication
fix-session-expiry-race-condition
refactor-extract-payment-service
chore-upgrade-node-20
docs-api-reference-update
```

**Rules:**

- Hyphens only — no underscores, no spaces, no slashes beyond the type prefix
- 3–5 words maximum in the description segment
- Include a ticket number when one exists: `feat-PROJ-123-oauth-integration`
- `main` (and `develop` if used) are protected — never commit directly to them

---

## Common Patterns

### Feature Branch Workflow

```bash
# Start from an up-to-date main
git checkout main && git pull

# Create a scoped feature branch
git checkout -b feat-PROJ-42-add-oauth-login

# Commit small, focused units of work as you go
git commit -m "feat(auth): add OAuth provider configuration"
git commit -m "feat(auth): implement OAuth callback handler"
git commit -m "test(auth): add unit tests for OAuth callback"

# Keep the branch current with main via rebase (not merge — avoids noise commits)
git fetch origin && git rebase origin/main

# Squash fixup / WIP commits before opening a PR
git rebase -i origin/main

# Push and open a PR
git push -u origin feat-PROJ-42-add-oauth-login
```

### Squashing Before Merge

```bash
# Interactive rebase onto the base branch
git rebase -i origin/main

# In the editor:
# - Keep the first meaningful commit as 'pick'
# - Mark cleanup/WIP commits as 'fixup' (discards their message)
#   or 'squash' (merges their message into the previous)
pick a1b2c3d feat(auth): add OAuth provider configuration
fixup d4e5f6a wip: debug callback
fixup 789abcd fix typo
```

### Reverting a Bad Commit

```bash
# Creates a new commit that undoes the changes — safe for shared history
git revert <sha>

# Commit message format:
# revert: feat(auth): add OAuth provider configuration
#
# Reverts commit abc1234.
# Reason: introduced regression in session token renewal — see #188.
```

---

## Anti-Patterns

### Meaningless commit messages

❌ Don't:

```
git commit -m "fix"
git commit -m "wip"
git commit -m "changes"
git commit -m "asdfasdf"
```

✅ Do:

```
git commit -m "fix(cart): correct item count when quantity is zero"
git commit -m "feat(checkout): add address validation on form submit"
```

### Bundling unrelated changes

❌ Don't:

```
git commit -m "fix login bug and update dependencies and add dark mode"
```

✅ Do:

```
git commit -m "fix(auth): prevent redirect loop on failed login"
git commit -m "chore: upgrade express to 4.19.0"
git commit -m "feat(ui): add dark mode toggle"
```

### Merging main into your feature branch

❌ Don't:

```bash
git checkout feat-my-feature
git merge main  # creates a noisy merge commit in your branch history
```

✅ Do:

```bash
git checkout feat-my-feature
git rebase origin/main  # replays your commits on top of main — clean history
```

---

## Tools and Commands

- `git log --oneline --graph --decorate` — Compact visual history with branch labels
- `git rebase -i HEAD~N` — Interactive rebase for the last N commits (squash, reorder, edit)
- `git commit --amend --no-edit` — Add staged changes to the last commit without editing the message (only on unpushed commits)
- `git bisect start / good <sha> / bad <sha>` — Binary search through history to find the exact commit that introduced a bug
- `commitlint` — Lint commit messages against the Conventional Commits specification in CI
- `semantic-release` — Automated versioning and changelog generation driven entirely by commit types

## References

- [Conventional Commits Specification](https://www.conventionalcommits.org/)
- [commitlint](https://commitlint.js.org/)
- [semantic-release](https://github.com/semantic-release/semantic-release)
- [Git Rebase documentation](https://git-scm.com/docs/git-rebase)
