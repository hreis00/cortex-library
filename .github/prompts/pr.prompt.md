---
name: pr
description: Runs the full pull request workflow — validates commits, pushes the branch, generates a structured PR title and description following Conventional Commits, and opens the PR via gh CLI.
agent: agent
model: [GPT-4.1 (copilot), Claude Sonnet 4.6 (copilot)]
tools: ["execute", "read"]
argument-hint: "Usage: /pr [<source-branch>] [to <target-branch>]. Defaults: source = current branch, target = main. Examples: /pr — /pr feat-my-feature to main — /pr feat-my-feature to develop"
---

# Pull Request Workflow

You are a senior engineer opening a pull request. Follow these steps in order. Stop and report clearly if any step fails — do not skip steps or assume success.

## Step 1 — Resolve branches

- `{{source}}`: the branch to open the PR from. If not provided, run `git branch --show-current`.
- `{{target}}`: the base branch to merge into. If not provided, use `main`.

Confirm both branch names exist before proceeding.

## Step 2 — Stage and commit any uncommitted changes

Run `git status`. If the working tree is clean, proceed to Step 3.

If there are uncommitted changes:

1. Run `git diff HEAD` and `git status --short` to understand what changed.
2. Infer a Conventional Commits message (`<type>(<scope>): <description>`) from the nature of the changes. Apply all rules from `.github/skills/git-conventional-commits/SKILL.md` — type selection, scope naming, imperative tense, ≤ 72 character header.
3. Stage all changes: `git add -A`.
4. Commit: `git commit -m "<derived message>"`.
5. If there are untracked files that appear to be build artifacts, dependencies, or generated output (e.g. `node_modules/`, `dist/`, `__pycache__/`, `*.pyc`), skip them — do not stage. If genuinely ambiguous, stage and commit everything.

Do not ask the user to write the commit message. Do not stop. Continue to Step 3.

## Step 3 — Validate commit messages

Run `git log origin/{{target}}..{{source}} --oneline` to list all commits on `{{source}}` that are not yet in `{{target}}`.

Every commit message **must** follow Conventional Commits format: `<type>(<scope>): <description>`. Apply the rules from `.github/skills/git-conventional-commits/SKILL.md`.

If any commit message is non-conformant, list each offending commit with the reason it fails, then stop. Suggest the fix: for the latest unpushed commit use `git commit --amend`; for multiple commits use `git rebase -i origin/{{target}}`.

## Step 4 — Push the branch

Run:

```bash
git push -u origin {{source}}
```

If the push fails (e.g. non-fast-forward), report the error verbatim and stop.

## Step 5 — Derive the PR title

The PR title must itself follow Conventional Commits: `<type>(<scope>): <description>`.

- If there is exactly one commit ahead, use that commit message as the title.
- If there are multiple commits, synthesize a single title that captures the overall change. Use the dominant `type` and `scope` from the commit log. The description must be in imperative present tense and ≤ 72 characters total.

## Step 6 — Write the PR description

Write the description using this exact structure. Omit any section that genuinely does not apply, but never omit **What**, **Why**, or **How**.

```
### What

<what changed — specific files, modules, or features; use bullets for multiple changes>

### Why

<motivation — what problem this solves or what requirement drove it; reference linked issues as `Closes #N` or `Refs #N`>

### How

<technical approach — non-obvious decisions, trade-offs, alternatives considered>

### Testing

<how the changes were verified — tests added/modified with what they cover, manual steps a reviewer can follow>

### Breaking Changes

<if any public API, config format, schema, or interface is not backward-compatible, list each change and its migration path>
```

Tone: factual and direct. Present tense. No filler phrases. No promotional language.

## Step 7 — Open the PR

Run:

```bash
gh pr create \
  --base {{target}} \
  --head {{source}} \
  --title "<derived title from Step 5>" \
  --body "<full description from Step 6>"
```

After the command succeeds, print the PR URL returned by `gh`.

---

## Variables

- `{{source}}` — Branch to open the PR from. Defaults to the current branch if omitted.
- `{{target}}` — Base branch to merge into. Defaults to `main` if omitted.

---

## Example

**Invocation:** `/pr feat-auth-setup to main`

**Step 2** — `git status` shows two modified files: `auth/config.py` and `auth/views.py`, and one new file `tests/test_oauth.py`. The agent infers `feat(auth): add OAuth login with provider configuration and callback`, runs `git add -A`, and commits. Working tree is now clean.

**Step 3** — `git log origin/main..feat-auth-setup --oneline` returns:

```
a1b2c3d feat(auth): add OAuth provider configuration
d4e5f6a feat(auth): implement OAuth callback handler
789abcd test(auth): add unit tests for OAuth callback
```

All messages conform to Conventional Commits. No action needed.

**Step 4** — `git push -u origin feat-auth-setup` succeeds.

**Step 5** — Three commits, all `feat(auth)`. Derived title: `feat(auth): add OAuth login with provider configuration and callback`

**Step 6** — Generated description:

```markdown
### What

- Added OAuth provider configuration in `auth/config.py`
- Implemented OAuth callback handler in `auth/views.py`
- Added unit tests covering successful callback, invalid state token, and provider error response

### Why

Password-based login was the only authentication option. OAuth allows users to sign in with existing provider accounts, reducing friction and eliminating password storage for federated users.

### How

The OAuth flow uses the Authorization Code grant. Provider credentials are read from environment variables — never hardcoded. The `state` parameter is a signed JWT to prevent CSRF on the callback.

`django-allauth` was considered but rejected; it brings significant model migrations and template overrides for a use case that only needs the callback flow.

### Testing

- `test_oauth_callback_success` — valid code + state returns 302 to dashboard, session contains user id
- `test_oauth_callback_invalid_state` — tampered state returns 400
- `test_oauth_callback_provider_error` — provider returns `error` param, user redirected to login with message
- Manual: full sign-in flow verified against GitHub OAuth app in development environment
```

**Step 7:**

```bash
gh pr create \
  --base main \
  --head feat-auth-setup \
  --title "feat(auth): add OAuth login with provider configuration and callback" \
  --body "..."
```

Output: `https://github.com/org/repo/pull/47`
