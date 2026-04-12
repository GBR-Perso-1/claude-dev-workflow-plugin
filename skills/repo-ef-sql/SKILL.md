---
name: repo-ef-sql
description: "Generate a SQL upgrade script from a given EF Core migration to the latest."
---

## Arguments

- `$ARGUMENTS` is the **full EF migration name** the database is currently on (i.e. the starting point, excluded from the script). Example: `20260320102621_2026-03-20`.
- Required. If empty, list available migrations and ask the user to pick one.

## Steps

### 1. Restore tools and list migrations

All `dotnet ef` commands must run from the `api/` directory (the tool manifest is at `api/.config/dotnet-tools.json`).

```bash
cd <repo-root>/api && dotnet tool restore && dotnet ef migrations list --project src/Infrastructure --startup-project src/Api --context ApplicationDbContext
```

Confirm that `$ARGUMENTS` appears in the list. If not, show all available migrations and stop.

### 2. Generate the script

```bash
cd <repo-root>/api && dotnet ef migrations script "$ARGUMENTS" --project src/Infrastructure --startup-project src/Api --context ApplicationDbContext -o ../sql-scripts/_latest-upgrade-db.sql
```

### 3. Report

Show:

- Starting point (excluded): the provided migration name
- **All migrations included** in the script (list each one)
- Output file: `sql-scripts/_latest-upgrade-db.sql`
- **Warning**: this script is **not idempotent** — running it twice on the same database will fail. If you need a safe-to-rerun version, ask again with the `--idempotent` flag.
