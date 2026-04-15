---
name: project-migrate-dev-settings
description: >
  Scan the repository for committed secrets and sensitive config files (default),
  or sanitise them from the working tree and full git history (sanitize argument).
---

## Important rules

Read and follow all rules in `${CLAUDE_PLUGIN_ROOT}/skills/shared/_ux-rules.md`.
Read and follow all rules in `${CLAUDE_PLUGIN_ROOT}/skills/shared/_git-rules.md`.
Read and follow the conventions in `${CLAUDE_PLUGIN_ROOT}/skills/shared/_dev-settings-conventions.md`.

## Arguments

`$ARGUMENTS` is optional.
- Empty / omitted → run **Scan & Report mode** (Epic 1)
- `"sanitize"` → run **Sanitise mode** (Epic 2)

Any other value: print usage and stop.

---

## Mode: Scan & Report (default)

### Phase 1 — Exclusion list

Before scanning, build the exclusion set:
- All paths reported by `git check-ignore --stdin` (gitignored paths)
- Binary files (detected by git's `--binary` flag or file extension heuristic)
- Known safe directories: `node_modules/`, `vendor/`, `.git/`, `dist/`, `build/`, `out/`, `bin/`, `obj/`

### Phase 2 — Three-strategy scan

Run all three detection strategies. Collect findings into a unified list.
Each finding records: file path, line number, detection method, redacted value preview,
candidate key name, target file suggestion, target variable suggestion.

**Strategy A — Pattern matching**

Scan all non-excluded, git-tracked text files for:
- High-entropy strings (≥ 20 chars of mixed alphanum/special)
- Known formats: Bearer tokens, connection strings containing `Password=`, `pwd=`, `password:`, AWS key patterns (`AKIA...`), private key PEM blocks (`-----BEGIN * PRIVATE KEY-----`)
- Common secret variable names assigned to non-empty string literals: `API_KEY`, `SECRET`, `PASSWORD`, `TOKEN`, `CLIENT_SECRET`, `PRIVATE_KEY`, `CONNECTION_STRING`, etc. (case-insensitive; right-hand side must be a non-empty string literal, not a placeholder)

**Strategy B — Committed env/config files**

Identify git-tracked files matching these name patterns:
`.env`, `.env.*`, `appsettings.*.json` (excluding `appsettings.json` itself unless it contains actual values), `secrets.json`, `local.settings.json`, `*.local.json`, `.secrets`, `credentials.*`

Flag each file as a finding if it is tracked by git.

**Strategy C — Manifest cross-reference**

If `.claude/dev-settings.json` exists, parse it. For each entry's key, attempt to retrieve its value from the private source (`it--dev-settings`) only if `gh` auth is available.
If `gh` is not available or not authenticated, skip this strategy and note it in the report.
For each value retrieved, grep all non-excluded tracked files for that literal value. Flag any match as a finding.

### Phase 3 — Report

Group findings by detection method. Show summary counts first.

For each finding:
- File path and line number
- Detection method
- Redacted preview (first 4 chars + `***` — never the full value)
- Suggested `dev-settings.json` entry (key name, targetFile, targetVariable)

Conclude with a **"What to do"** section listing keys to add to `it--dev-settings` and directing the developer to run `/project-migrate-dev-settings sanitize` to act.

If no findings: report clean with scan summary (files scanned, strategies used).

---

## Mode: Sanitise (`sanitize` argument)

### Phase 4 — Full scan

Re-run the three-strategy scan (same as Scan & Report Phase 2).
If zero findings: halt with a clean message.

### Phase 5 — Per-secret review

For each finding, present it individually via `AskUserQuestion`:
- Show: file, line, detection method, redacted preview, proposed manifest entry
- Options:
  1. "Accept — use suggested entry"
  2. "Edit — modify key / targetFile / targetVariable before accepting"
  3. "Skip — exclude this finding (false positive)"
- If "Edit": collect corrected values; validate that key has ≥ 3 dot-separated segments, targetFile is non-empty, targetVariable is non-empty.
- Track: approved findings (with their manifest entries) and skipped findings.

If all findings are skipped: halt with a message. No changes made.

### Phase 6 — Working tree changes (autonomous after review)

For each approved finding:

- **REQ-2.4**: Replace the exposed secret value in the source file with a reference to the target variable (e.g. for dotenv-style files: replace value with `${TARGET_VARIABLE}`; for JSON: replace with `"<injected>"`).
- **REQ-2.5**: For Strategy B findings (committed env/config files): remove the secret value from the file (replace the variable line with a blank value or placeholder comment) and append the file path to `.gitignore`.

**REQ-2.6**: Create or update `.claude/dev-settings.json` with all approved manifest entries. Existing entries are preserved; new entries are appended. Use the manifest format from `_dev-settings-conventions.md`.

**REQ-2.7**: For each `targetFile` in new manifest entries: check `.gitignore` and add the path if absent.

### Phase 7 — Confirmation gate (before history rewrite)

Present a summary:
- Number of commits containing the secrets (count and earliest commit SHA and date)
- Affected files and variable names
- Clear warning: history rewrite will require a force push and will invalidate collaborators' local history

Use `AskUserQuestion`:
- "Rewrite git history and force-push (irreversible)"
- "Cancel history rewrite — keep working tree changes"

If "Cancel": stop. Keep all working tree changes. Skip to Phase 10.

### Phase 8 — History purge

**REQ-2.11**: For each approved secret value, purge it from the full git history across all locally checked-out branches.

Preferred tool: `git filter-repo`. Check availability first. Fallback: BFG Repo Cleaner. If neither is available: halt with installation instructions and error message. Do not retry.

**REQ-2.12**: After rewrite, ask the developer to confirm the branch name and remote before pushing. Use `AskUserQuestion`. On confirmation: run `git push --force-with-lease <remote> <branch>` for each rewritten branch.

**REQ-2.13**: If rewrite or push fails: surface the exact error and halt. Do not retry.

### Phase 9 — Private source handoff

**REQ-2.14**: Display the list of keys to add to `it--dev-settings`. Include key naming convention reminder (from `_dev-settings-conventions.md`) and instructions to contact the platform admin (Guillaume).

**REQ-2.15**: Use `AskUserQuestion`: "Have you added all keys to `it--dev-settings`?"
- "Yes — all keys are added"
- "Not yet — I'll do it later"

If "Not yet": note it in the final summary. Skip REQ-2.16.

**REQ-2.16**: If confirmed: offer to run `/project-inject-dev-settings`.
- "Yes — run `/project-inject-dev-settings` now"
- "No — I'll run it manually later"

### Phase 10 — Final summary

**REQ-2.17**: Print final summary:
- Findings processed vs skipped (counts)
- Files modified in working tree (list)
- Git history rewritten (yes/no) — if yes, list branches rewritten
- Branches force-pushed (list or "none")
- Injection run (yes/no)

**REQ-2.18**: For every secret found in git history: print **ROTATE SECRETS** warning:
> "The following secrets were found in git history. Any clone made before the rewrite retains access. You must rotate these credentials immediately:"
> List each affected variable name and file.

---

## Guardrails

- Never print, echo, log, or include any secret value in any output — not in summaries, error messages, previews, or tool-call commentary.
- Never stage, commit, or push target files containing injected values.
- Never write to `it--dev-settings`.
- Never purge secrets from branches not checked out locally.
- Always use `git push --force-with-lease` — never bare `--force`.
- Never skip the Phase 7 confirmation gate, even if the user says "just do it".
- All `AskUserQuestion` gates must offer a cancel/skip path.
