---
name: project-init
description: "Bootstrap or refresh a project's .claude/ directory. First run: full setup (CLAUDE.md, project-context.md, rules). Subsequent runs: refresh rules from the plugin."
---

Bootstrap or refresh a project's `.claude/` directory for use with this Claude Code plugin.

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Mode Detection

Check if `.claude/CLAUDE.md` already exists.

- **Does not exist** → run **Full Bootstrap** (all steps below)
- **Already exists** → run **Refresh Rules Only** (skip to step 5)

---

## Full Bootstrap

### 1. Gather project information

Auto-detect the stack by checking which directories exist in the project root (`api/`, `app/`, `infra/`, `**/*.py`).

Use `AskUserQuestion` to collect and confirm:

1. **Project name** — the repository/project name (e.g. `fi--castellan`)
2. **Description** — one-sentence description of what the project does
3. **Stack** — present the auto-detected subprojects as pre-selected defaults. Options:
   - `.NET API` (has `api/` directory)
   - `Vue Frontend` (has `app/` directory)
   - `Terraform Infrastructure` (has `infra/` directory)
   - `Python` (has Python files)
4. **Team size** — number of developers (default: 1)
5. **User base** — e.g. "internal tool, ~50 users" or "public-facing"

### 2. Create `.claude/` directory

```bash
mkdir -p .claude
```

### 3. Create `.claude/CLAUDE.md`

Write a thin CLAUDE.md tailored to the gathered information:

```markdown
# <project-name>

<description>

@./project-context.md

## Structure

<list detected subprojects based on stack, e.g.:>
- `api/` — .NET backend API
- `app/` — Vue 3 frontend
- `infra/` — Terraform infrastructure

## Deep Context

Run `/project-session` at the start of a session to load full architectural context (commands, patterns, stack conventions). Skip it for quick edits or focused tasks.

## Guardrails

- Never push to remote without user confirmation
- Never edit synced rule files in `.claude/rules/` — managed by `/project-init`
<if infra/ detected:>
- Never run destructive Terraform commands without explicit confirmation
<if api/ detected:>
- Never modify authentication code without explicit confirmation
- Never edit auto-generated API client files manually — regenerate from source
```

Adapt the guardrails to the detected stack — only include relevant ones.

### 4. Copy and fill `project-context.md`

Copy the template from the plugin:

```bash
cp "${CLAUDE_PLUGIN_ROOT}/templates/project-context.md" .claude/project-context.md
```

Then update the copied file with the values gathered in step 1:
- **Company**: from step 1 (the project/company name the user provided)
- **Team size**: from step 1
- **User base**: from step 1
- **Ownership model**: developers own full stack including infra
- **Release cadence**: continuous to main

### 5. Create `.claude/settings.json`

Only during **Full Bootstrap**. Skip if `.claude/settings.json` already exists.

Write `.claude/settings.json` with `enabledPlugins` for the detected stack:

```json
{
  "enabledPlugins": {
    <if api/ detected:>
    "csharp-lsp@claude-plugins-official": true,
    <if app/ detected:>
    "typescript-lsp@claude-plugins-official": true,
    <if Python files detected:>
    "pyright-lsp@claude-plugins-official": true
  }
}
```

Include only entries relevant to the detected stack. If no stack-specific LSPs are needed, the `enabledPlugins` object can be empty or omitted. Do not add plugin names here — plugins are managed at the user or workspace level, not per-project.

### 7. Sync rules

This step runs in both **Full Bootstrap** and **Refresh Rules Only** modes.

Auto-detect the stack (check for `api/`, `app/`, `infra/`, `**/*.py`).

```bash
mkdir -p .claude/rules
```

**Always copy:**
- `${CLAUDE_PLUGIN_ROOT}/rules/general.md` → `.claude/rules/general.md`

**Copy conditionally:**

| Detection | Rule files to copy |
|---|---|
| `api/` exists | `api-lang.md` |
| `app/` exists | `app-lang.md` |
| `infra/` exists | `infra-lang.md`, `infra-naming.md` |
| Any `**/*.py` files exist | `python-lang.md` |

Overwrite existing rule files — the plugin is the source of truth.

### 8. Summary

**Full Bootstrap:**
```
Created:
  .claude/CLAUDE.md          — project identity and session start instruction
  .claude/project-context.md — project context (edit to refine)
  .claude/settings.json      — enabled plugins for this project's stack
  .claude/rules/             — engineering rules (synced from plugin)

Rules synced: general.md, api-lang.md, app-lang.md (based on detected stack)
Plugins enabled: csharp-lsp, typescript-lsp (based on detected stack)

Next steps:
  - Review and edit .claude/project-context.md with project-specific details
  - Run /project-session at the start of a session to load full architectural context
```

**Refresh Rules Only:**
```
Rules synced: general.md, api-lang.md, app-lang.md (based on detected stack)

All rules refreshed from plugin. No other files were modified.
```
