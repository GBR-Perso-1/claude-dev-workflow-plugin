---
name: project-inject-dev-settings
description: "Fetch dev credentials from the private Rise settings source and inject them into local config files declared in .claude/dev-settings.json."
---

## Important rules

Read and follow all rules in `${CLAUDE_PLUGIN_ROOT}/skills/shared/_ux-rules.md`.

Read and follow the conventions in `${CLAUDE_PLUGIN_ROOT}/skills/shared/_dev-settings-conventions.md`.

## Guardrails

- **Never** reveal any credential value in any form — not in summaries, error messages, step narration, reasoning output, or tool-call commentary. The value exists only inside `$NEWVAL` in the shell. Do not refer to, quote, or describe the content of `$NEWVAL` at any point.
- **Never** stage, commit, or push any target file.
- **Never** modify `.claude/dev-settings.json`.
- **Only** write to files declared in the manifest — no other files may be touched.

---

## Phase 0 — Preflight

All checks run before anything else. Halt on first failure.

### Step 0.1 — Check `jq` is installed

```bash
jq --version
```

If absent, halt:

```
jq is required but not installed.
Install it: winget install jqlang.jq (Windows) / brew install jq (macOS)
```

### Step 0.2 — Check `gh` is authenticated

```bash
gh auth status
```

If not authenticated, halt:

```
gh CLI is not authenticated. Run: gh auth login
```

### Step 0.3 — Verify private source repo is reachable

```bash
gh repo view it--dev-settings --json name --jq '.name' 2>&1
```

On any failure, halt:

```
Cannot reach it--dev-settings. Check your gh authentication and network connection.
If you are a new team member, you may need to request access from the platform admin.
```

### Step 0.4 — Verify manifest exists

```bash
test -f .claude/dev-settings.json && echo "EXISTS" || echo "MISSING"
```

If missing, halt:

```
No manifest found at .claude/dev-settings.json.
This repo has not declared its dev settings requirements.
```

### Step 0.5 — Parse and validate manifest

Parse `.claude/dev-settings.json`. It must be a JSON array where each entry contains `key`, `targetFile`, and `targetVariable`. If malformed or missing required fields, halt with the parse error details and the names of any entries that fail validation.

---

## Phase 1 — Fetch private source

### Step 1.1 — Download settings to a temp file

```bash
SETTINGS_FILE="/tmp/it--dev-settings-$(date +%s)_$$.json"
gh api repos/it--dev-settings/contents/settings.json \
  --jq '.content' | base64 -d > "$SETTINGS_FILE"
```

Note: on macOS, replace `base64 -d` with `base64 -D` if you see a "invalid option" error.

If this command fails for any reason, delete the temp file and halt:

```bash
rm -f "$SETTINGS_FILE"
```

```
Failed to fetch settings from it--dev-settings. No values were injected.
```

### Step 1.2 — Verify all keys are present

For every `key` in the manifest, verify it exists in `$SETTINGS_FILE`. Collect all missing keys. If any are missing, delete the temp file and halt:

```bash
rm -f "$SETTINGS_FILE"
```

```
The following keys were not found in the private settings source:
  - <key1>
  - <key2>

Ask the platform admin (Guillaume) to add these entries.
No values were injected.
```

---

## Phase 2 — .gitignore guard

### Step 2.1 — Check each target file against `.gitignore`

For each unique `targetFile` in the manifest, set `$TARGET_FILE` to that value and run:

```bash
grep -Fx "$TARGET_FILE" .gitignore
```

where `$TARGET_FILE` is the `targetFile` value from the current manifest entry being iterated. If not found, append the path to `.gitignore`. Collect all additions.

### Step 2.2 — Report additions

If any files were added to `.gitignore`, report:

```
.gitignore updated — the following injection targets were not gitignored and have been added:
  - <targetFile1>
  - <targetFile2>
```

### Step 2.3 — Continue

Proceed to injection without interruption regardless of whether any additions were made.

---

## Phase 3 — Injection

```bash
INJECT_TMP="/tmp/inject-json-tmp-$(date +%s)_$$.json"
```

For each entry in the manifest:

### Step 3.1 — Create target file if absent

If `targetFile` does not exist:
- If its extension is `.json`: create it with the content `{}` (valid empty JSON object).
- Otherwise: create it as an empty file.

### Step 3.2 — Determine write strategy

- `.json` extension → JSON upsert (Step 3.4)
- All other extensions → dotenv upsert (Step 3.3)

### Step 3.3 — Dotenv upsert

```bash
NEWVAL=$(jq -r '.[] | select(.key=="<key>") | .value' "$SETTINGS_FILE")
if grep -q "^<targetVariable>=" "<targetFile>"; then
  sed -i.bak "s|^<targetVariable>=.*|<targetVariable>=$NEWVAL|" "<targetFile>" && rm "<targetFile>.bak"
else
  echo "<targetVariable>=$NEWVAL" >> "<targetFile>"
fi
```

### Step 3.4 — JSON upsert

```bash
NEWVAL=$(jq -r '.[] | select(.key=="<key>") | .value' "$SETTINGS_FILE")
jq --arg v "$NEWVAL" '.<targetVariable> = $v' "<targetFile>" > "$INJECT_TMP" \
  && mv "$INJECT_TMP" "<targetFile>"
```

---

## Phase 4 — Cleanup and summary

### Step 4.1 — Delete temp files

```bash
rm -f "$SETTINGS_FILE"
rm -f "$INJECT_TMP"
```

### Step 4.2 — Print summary

Never print any credential value in the summary.

```
Inject Dev Settings — Complete

Files written:
  <targetFile>
    <targetVariable>   [set]
    ...

These files are local-only and must not be committed to source control.
Verify with: git status
```

If `.gitignore` additions were made during Phase 2, include them in the summary:

```
.gitignore additions:
  - <targetFile1>
  - <targetFile2>
```
