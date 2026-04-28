---
name: project-init
description: "Bootstrap or refresh a project's .claude/ directory. First run: full setup (CLAUDE.md, project-context.md, rules). Subsequent runs: refresh rules from the plugin."
---

Bootstrap or refresh a project's `.claude/` directory for use with this Claude Code plugin.

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

When running in Full Bootstrap mode, also read and follow the conventions in [`../shared/_dev-settings-conventions.md`](../shared/_dev-settings-conventions.md).

## Mode Detection

Check if `.claude/CLAUDE.md` already exists.

- **Does not exist** → run **Full Bootstrap** (all steps below)
- **Already exists** → run **Refresh Rules Only** (skip to step 7). Dev settings are not checked, mentioned, or modified in this mode.

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

- **NEVER** modify Azure without explicit confirmation, you can read but NEVER WRITE to Azure without confirmation
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

### 6. Dev Settings

This step runs in **Full Bootstrap** mode only. It is skipped entirely in Refresh Rules Only mode.

#### 6A — Check for existing manifest

```bash
test -f .claude/dev-settings.json && echo "EXISTS" || echo "MISSING"
```

Branch on the result:

- **MISSING** → proceed to **6B** (manifest wizard)
- **EXISTS** → proceed to **6C** (offer injection)

---

#### 6B — Manifest wizard (no manifest found)

Use `AskUserQuestion` to ask:

> Does this project require injected dev credentials (API keys, secrets, connection strings)?

Options:
1. Yes — help me create the manifest now (Recommended)
2. No — this project does not use injected credentials
3. Skip for now — I'll set this up later

If the user selects **No**: do not create any file. Record outcome as `manifest_declined`. Proceed to Step 7.

If the user selects **Skip for now**: do not create any file. Record outcome as `manifest_skipped`. Proceed to Step 7.

If the user selects **Yes**: begin the entry collection loop below.

**Entry collection loop**

Collect one manifest entry at a time. For each entry, present a prompt via `AskUserQuestion`:

> Entry #{n} — select an action:

Options:
1. Add an entry
2. Done — save manifest and continue
3. Cancel — discard all entries

If the user selects **Done** and no entries have been collected yet: display the message "At least one entry is required to save a manifest. Add an entry or select Cancel." then re-present the Add / Done / Cancel `AskUserQuestion`. Highlight Cancel as the escape option for users who do not want to define any entries. Do not save an empty manifest.

If the user selects **Done** with one or more valid entries: save the manifest (see below).

If the user selects **Cancel**: discard all collected entries, do not create any file. Record outcome as `manifest_skipped`. Proceed to Step 7.

If the user selects **Add an entry**: collect the three fields in a free-text exchange:

- `key` — the setting key following `<org>.<repo>.<purpose>` convention (e.g. `rise.wealth-pilot.azure-client-secret`)
- `targetFile` — relative path from the repo root to the file that receives the value (e.g. `api/src/.env.local`)
- `targetVariable` — variable name inside that file (e.g. `AZURE_CLIENT_SECRET`)

**Validation** — before accepting the entry:

- `key` must be non-empty and contain at least three dot-separated segments (e.g. `a.b.c`). If invalid: "Key must follow the `<org>.<repo>.<purpose>` format (e.g. `rise.wealth-pilot.azure-client-secret`)." Re-prompt the key field.
- `targetFile` must be non-empty. If empty: "Target file path cannot be empty. Please provide a relative path from the repo root (e.g. `api/src/.env.local`)." Re-prompt the targetFile field.
- `targetVariable` must be non-empty. If empty: "Target variable cannot be empty. Please provide a variable name (e.g. `AZURE_CLIENT_SECRET`)." Re-prompt the targetVariable field.

Only accept the entry once all three fields pass validation. After accepting, return to the top of the loop (show the add/done/cancel options again for entry #{n+1}).

**Save manifest**

Write the collected entries as a JSON array to `.claude/dev-settings.json`. Each object must have exactly: `key`, `targetFile`, `targetVariable` — no other fields. Use Claude's file-write capability (not shell redirection) to write the JSON.

Record outcome as `manifest_created`. Do **not** attempt injection after saving. Proceed to Step 7.

---

#### 6C — Offer injection (manifest already exists)

Parse `.claude/dev-settings.json`. Validate it is a JSON array where each entry has non-empty `key`, `targetFile`, and `targetVariable`. If invalid, report the parse error and record outcome as `manifest_invalid`. Proceed to Step 7 without offering injection.

If valid, use `AskUserQuestion` to ask:

> `.claude/dev-settings.json` found with {n} entr{y/ies}. Would you like to inject dev credentials now?

Options:
1. Yes — inject credentials now (Recommended)
2. No — I'll run /project-inject-dev-settings later

If the user selects **No**: record outcome as `injection_deferred`. Proceed to Step 7.

If the user selects **Yes**: run the **Embedded Injection Flow** below.

**Embedded Injection Flow**

> **Maintenance note**: this flow mirrors `/project-inject-dev-settings`. If that skill's phases change (including Phase 0 / Step 0.0 dev-settings source resolution), update this embedded copy in sync.

All guardrails apply without exception:

- **Never** reveal any credential value in any form — not in summaries, error messages, step narration, reasoning output, or tool-call commentary.
- **Never** stage, commit, or push any target file.
- **Never** modify `.claude/dev-settings.json`.
- **Only** write to files declared in the manifest.

On any halt condition below, record outcome as `injection_failed` with the halt reason and continue to Step 7 (do not abort project-init).

> On any halt condition, run `rm -f "$SETTINGS_FILE" "$INJECT_TMP"` before recording the outcome and proceeding to Step 7.

**Phase 0 — Resolve dev-settings source**

Apply R.2–R.3 from the context resolution contract against the current working directory to resolve the active context.

- If a context is resolved and it declares `dev_settings_repo` and `dev_settings_owner`:
  ```
  DEV_SETTINGS_REPO="<dev_settings_owner>/<dev_settings_repo>"
  ```
- If the context is resolved but one or both fields are absent, or if no context matches and no manifest exists:
  ```
  DEV_SETTINGS_REPO="it--dev-settings"
  ```
  Announce the fallback: `No dev_settings_repo/dev_settings_owner found in manifest context — falling back to unqualified it--dev-settings.`

**Phase I — Preflight**

Step I.1 — Check `jq` is installed:
```bash
jq --version
```
If absent: record outcome as `injection_failed` with reason `jq is required but not installed. Install it: winget install jqlang.jq (Windows) / brew install jq (macOS)`, then proceed to Step 7.

Step I.2 — Check `gh` is authenticated:
```bash
gh auth status
```
If not authenticated: record outcome as `injection_failed` with reason `gh CLI is not authenticated. Run: gh auth login`, then proceed to Step 7.

Step I.3 — Verify private source repo is reachable:
```bash
gh repo view $DEV_SETTINGS_REPO --json name --jq '.name' 2>&1
```
On any failure: record outcome as `injection_failed` with reason `Cannot reach $DEV_SETTINGS_REPO. Check your gh authentication and network connection. If you are a new team member, you may need to request access from the platform admin.`, then proceed to Step 7.

Step I.4 — Manifest is already validated in 6C above. Skip re-validation.

**Phase II — Fetch private source**

Step II.1 — Download settings to a temp file:
```bash
SETTINGS_FILE="/tmp/dev-settings-$(date +%s)_$$.json"
gh api repos/$DEV_SETTINGS_REPO/contents/settings.json \
  --jq '.content' | base64 -d > "$SETTINGS_FILE"
```
Note: on macOS, replace `base64 -d` with `base64 -D` if you see an 'invalid option' error.

If this fails: delete the temp file, record outcome as `injection_failed` with reason `Failed to fetch settings from $DEV_SETTINGS_REPO. No values were injected.`, then proceed to Step 7.

Step II.2 — Verify all keys are present. For every `key` in the manifest, verify it exists in `$SETTINGS_FILE`. Collect missing keys. If any missing: delete temp file, record outcome as `injection_failed` with reason:
```
The following keys were not found in the private settings source:
  - <key1>
  - <key2>

Ask the platform admin to add these entries.
No values were injected.
```
Then proceed to Step 7.

**Phase III — .gitignore guard**

Step III.1 — For each unique `targetFile` in the manifest, set `$TARGET_FILE` to that value and run:
```bash
grep -Fx "$TARGET_FILE" .gitignore
```
If not found, append the path to `.gitignore`. Collect all additions.

Step III.2 — If any files were added to `.gitignore`, note them for the Step 8 summary. Continue without interruption.

**Phase IV — Injection**

```bash
INJECT_TMP="/tmp/inject-json-tmp-$(date +%s)_$$.json"
```

For each entry in the manifest:

Step IV.1 — Create target file if absent:
- `.json` extension → create with `{}`
- Otherwise → create as empty file

Step IV.2 — Determine write strategy:
- `.json` extension → JSON upsert (Step IV.4)
- All other extensions → dotenv upsert (Step IV.3)

Step IV.3 — Dotenv upsert:
```bash
NEWVAL=$(jq -r '.[] | select(.key=="<key>") | .value' "$SETTINGS_FILE")
if grep -q "^<targetVariable>=" "<targetFile>"; then
  sed -i.bak "s|^<targetVariable>=.*|<targetVariable>=$NEWVAL|" "<targetFile>" && rm "<targetFile>.bak"
else
  echo "<targetVariable>=$NEWVAL" >> "<targetFile>"
fi
```

Step IV.4 — JSON upsert:
```bash
NEWVAL=$(jq -r '.[] | select(.key=="<key>") | .value' "$SETTINGS_FILE")
jq --arg v "$NEWVAL" '.<targetVariable> = $v' "<targetFile>" > "$INJECT_TMP" \
  && mv "$INJECT_TMP" "<targetFile>"
```

**Phase V — Cleanup**

```bash
rm -f "$SETTINGS_FILE"
rm -f "$INJECT_TMP"
```

Record outcome as `injection_complete`. Note the list of files written and any `.gitignore` additions for Step 8 summary.

### 7. Sync rules

This step runs in both **Full Bootstrap** and **Refresh Rules Only** modes.

Auto-detect the stack (check for `api/`, `app/`, `infra/`, `**/*.py`). Reuse the stack detection result from Step 1 — do not re-run directory checks if this follows a Full Bootstrap.

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
```

Append a **Dev Settings** block based on the outcome recorded in Step 6:

| Outcome | Dev Settings line to append |
|---------|----------------------------|
| `manifest_declined` | *(omit — print nothing)* |
| `manifest_skipped` | `Dev Settings: skipped. Run /project-init again or create .claude/dev-settings.json manually when ready.` |
| `manifest_created` | `Dev Settings: .claude/dev-settings.json created ({n} entries). No credentials were injected. Run /project-inject-dev-settings once the keys have been added to the private source.` |
| `manifest_invalid` | `Dev Settings: .claude/dev-settings.json exists but could not be parsed — see error above. Run /project-inject-dev-settings after correcting the manifest.` |
| `injection_deferred` | `Dev Settings: .claude/dev-settings.json found. Run /project-inject-dev-settings to hydrate the local environment.` |
| `injection_complete` | `Dev Settings: credentials injected into {list of target files}. These files are local-only and must not be committed to source control.` + if gitignore additions were made: `.gitignore updated: {list of files added}` |
| `injection_failed` | `Dev Settings: injection attempted but failed — {halt reason}. Run /project-inject-dev-settings to retry once the issue is resolved.` |

**Next steps:**
```
  - Review and edit .claude/project-context.md with project-specific details
  - Run /project-session at the start of a session to load full architectural context
```

Include this additional line only when outcome is `manifest_created`:
```
  - Run /project-inject-dev-settings once the keys have been added to the private dev-settings source
```

Include this additional line only when outcome is `injection_deferred`:
```
  - Run /project-inject-dev-settings to hydrate your local environment with dev credentials
```

---

**Refresh Rules Only:**

```
Rules synced: general.md, api-lang.md, app-lang.md (based on detected stack)

All rules refreshed from plugin. No other files were modified.
```

No dev settings line is appended in this mode.
