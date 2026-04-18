---
name: repo-commit
description: "Run tests, Commit and push changes to the current branch."
---

## Arguments

- `$ARGUMENTS` is **optional**. Scope the commit to a subproject: `api`, `app`, `infra`.
- If empty, commit **all changes** across the whole repo.

**Examples:**

- `/repo-commit` → commit everything
- `/repo-commit api` → commit only changes under `api/`
- `/repo-commit app` → commit only changes under `app/`

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

Read and follow the context resolution contract in [`../shared/_context-resolution.md`](../shared/_context-resolution.md).

## Process

### 1. Run tests

Run tests for the affected subproject(s). If any test fails, report the failures and **stop** — do not proceed.

- `api`: `dotnet test api/{{solutionName}}.sln`
- `app`: `npm run --prefix app test`
- `infra`: skip tests
- Unscoped: run tests for all subprojects that exist and have changes

### 2. Stage and generate commit message

Stage changes:

- If scoped: `git add <subproject>/`
- If unscoped: `git add` only the changed files shown in step 1. Never use `git add -A`.

Run `git diff --cached` to review the staged diff. Generate a conventional commit message:

- Format: `type(scope): description` (e.g. `feat(auth): add token refresh`, `fix(api): handle null value`)
- Short description (under 72 chars), lowercase, no period
- Multi-line body with bullet points if multiple distinct changes
- Always append `Co-Authored-By: Claude <noreply@anthropic.com>`

### 3. Commit

Create the commit with the message.

### 4. Pull and rebase

```bash
git fetch origin
git pull --rebase
```

If conflicts occur, abort the rebase (`git rebase --abort`) and **stop**. Tell the user their commit is saved locally but conflicts need manual resolution.

### 4a. Context consistency check (before push)

Apply R.2–R.3 from the context resolution contract against the current working directory:

1. Read `%USERPROFILE%\.claude\contexts.json` (Windows) / `$HOME/.claude/contexts.json` (Unix). If absent, skip this step and proceed to Step 5 with existing behaviour.
2. Match cwd against `path_globs` in manifest order (first match wins).
3. If a context matches, extract the owner from `git remote get-url origin` and compare it to the context's `github_org` or `github_user`.
   - **Match**: proceed silently to Step 5.
   - **Mismatch**: invoke the R.3 signal-mismatch path — report and ask via `AskUserQuestion`. Do **not** push automatically. Options:
     1. Push anyway — I've verified this is correct
     2. Cancel push — I'll fix the remote first
   - The "save to manifest" opt-in is offered **at most once per push** if the user had to answer inline.
4. If no context matches: proceed to Step 5 with existing behaviour (no context check).

### 5. Push

Show the user:

- The commit message
- The target branch
- Whether any remote changes were rebased in

**GATE** — Ask if the user wants to push. On confirmation:

```bash
git push
```

Never force-push. If push fails, report the error and stop.

## Output

```
Committed: <commit-message>
Pushed to: <branch-name>
```
