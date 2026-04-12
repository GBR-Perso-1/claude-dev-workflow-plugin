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
