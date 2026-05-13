---
name: repo-tidy
description: "Clean up the project's `.claude/settings.json` and `.claude/settings.local.json` — remove session artifacts, group by category, sort within groups, and consolidate redundant entries."
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Steps

### 1. Read settings files

From the project root, read `.claude/settings.json` and `.claude/settings.local.json` (if it exists).

### 2. Analyse each `allow` entry

For each entry, classify it:

- **Session artifact** — a long one-time bash command auto-approved during a past session. Signs: heredocs (`<< 'EOF'`), `while/do/done` fragments, `find` piped to temp files, `cat > /tmp/...`, multi-line strings with `\n`, specific one-off debugging invocations. **Remove.**
- **Project-specific leak** — a `Read(...)` or `Bash(...)` scoped to a different project path. **Remove.**
- **Keeper** — a genuine, reusable permission. **Keep.**

### 3. Consolidate keepers

Group keepers into categories and sort alphabetically within each group:

1. **System** — `curl`, `find`, `grep`, `ls`, `netstat`, `pwd`, `taskkill`, `tasklist`, `test`, `echo`
2. **Azure / Terraform** — `az *`, `terraform *`
3. **dotnet** — all `dotnet *` subcommands
4. **npm / Node** — `npm *`, `npx *`, `nswag *`, env-prefixed variants (`VITE_*`)
5. **git (direct)** — `git add*`, `git branch*`, etc. (no `-C`)
6. **git (-C)** — `git -C *add*`, `git -C *branch*`, etc.
7. **Read permissions** — file/directory read grants, sorted by path
8. **Web** — `WebFetch`, `WebSearch`

**Wildcard consolidation rules:**

- Prefer trailing `*` without colon — e.g. `Bash(git fetch*)` over `Bash(git fetch:*)`.
- Do NOT collapse all `dotnet *` into one entry — subcommand granularity is intentional.
- Do NOT collapse all `git *` into one entry — granularity is intentional.
- Do collapse near-duplicates that differ only in suffix style (e.g. `git pull:*` and `git pull *` → keep `git pull*`).

### 4. Present proposed changes

For each file that has changes, show:

```
### .claude/settings.json

Removed (session artifacts):
  - Bash(find "C:\\..." ...)
  - Bash(while IFS= ...)

Removed (project-specific leak):
  - Read(//c/.../<other-project>/...)

Consolidated:
  - Bash(git fetch:*) → Bash(git fetch*)

Result: N entries → M entries
```

Then show the **full proposed JSON** so the user can review it.

### 5. Ask for confirmation

Use `AskUserQuestion` with options: `Apply`, `Edit first`, `Skip`.

### 6. Write

On confirmation, write the cleaned JSON. Use 2-space indentation. Preserve the structure:

```json
{
  "permissions": {
    "allow": [...]
  }
}
```

Do NOT commit.
