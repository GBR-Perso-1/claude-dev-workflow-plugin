---
name: project-migrate-dev-settings
description: >
  Scan the repository for committed secrets and sensitive config files (default),
  or sanitise them from the working tree and full git history (sanitize argument).
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).
Read and follow all rules in [`../shared/_git-rules.md`](../shared/_git-rules.md).
Read and follow the conventions in [`../shared/_dev-settings-conventions.md`](../shared/_dev-settings-conventions.md).

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
- Known formats: Bearer tokens, connection strings containing `Password=`, `pwd=`, `password:`, AWS key patterns (`AKIA...`), private key PEM blocks (`-----BEGIN * PRIVATE KEY-----`), API scope URIs matching `api://<guid>` patterns
- Common secret variable names assigned to non-empty string literals (case-insensitive; right-hand side must be a non-empty string literal, not a placeholder):
  - Credentials: `API_KEY`, `SECRET`, `PASSWORD`, `TOKEN`, `CLIENT_SECRET`, `PRIVATE_KEY`, `CONNECTION_STRING`, `APIKEY`
  - Azure / Entra: `ClientId`, `TenantId`, `TenantName`, `AppId`, `Audience`, `ClientSecret`, `Authority`
  - Generic: `ACCESS_KEY`, `SIGNING_KEY`, `ENCRYPTION_KEY`, `WEBHOOK_SECRET`

**Strategy B — Committed env/config files**

Identify git-tracked files matching these name patterns:
`.env`, `.env.*`, `appsettings.*.json` (excluding `appsettings.json` itself unless it contains actual values), `secrets.json`, `local.settings.json`, `*.local.json`, `.secrets`, `credentials.*`

For each matching file that is tracked by git, parse its contents and produce **one finding per variable**, not one finding per file. Do not filter by perceived sensitivity — every variable that is not a structural placeholder gets a finding, regardless of whether it looks like a secret. The purpose of the manifest is centralised rotation, not only security: even values like Entra ClientId or TenantId belong in the manifest.

- **dotenv-style files** (`.env`, `.env.*`): each `KEY=value` line where value is non-empty and not a placeholder is one finding.
- **JSON config files** (`appsettings.*.json`, `secrets.json`, etc.): enumerate every leaf string value at every nesting depth. Use the following `jq` command to extract all leaves programmatically — do not rely on manual inspection:
  ```
  jq -r 'path(..) as $p | select(getpath($p) | type == "string" and length > 0) | [($p | map(tostring) | join(":")), getpath($p)] | @tsv' <file>
  ```
  Each output row is one finding. The first column is the colon-joined JSON path (e.g. `AzureEntraId:ClientId`, `ConnectionStrings:SqlConnection`); the second column is the value (used only for redaction preview — never print it).

  **Example**: a file containing `{ "AzureEntraId": { "TenantId": "5dcc...", "ClientId": "3d3e...", "Audience": "api://3d3e..." } }` MUST produce three findings: `AzureEntraId:TenantId`, `AzureEntraId:ClientId`, `AzureEntraId:Audience`. If the jq command produces them, they are findings — do not discard any.

- Skip only values that are structural placeholders: empty strings, `null`, `<your-value>`, `placeholder`, `TODO`, `changeme`, `false`, `true`, `0`.
- Record the file path and line number for each finding.

**`targetFile` for Strategy B findings** — the manifest `targetFile` must point to the **gitignored injection target**, not the committed source file. The committed source file will have its values replaced with placeholders and stay in git as a template; the injection skill writes real values into the separate target file. Use these conventions:

| Committed source file | Suggested injection target (`targetFile`) |
|----------------------|-------------------------------------------|
| `.env` | `.env.local` |
| `.env.<name>` | `.env.<name>.local` |
| `appsettings.<env>.json` | `appsettings.local.json` |
| `secrets.json` | `secrets.local.json` |
| `local.settings.json` | `local.settings.local.json` |

The injection target file does not need to exist yet — the injection skill creates it.

**Strategy C — Manifest cross-reference**

If `.claude/dev-settings.json` exists, parse it. For each entry's key, attempt to retrieve its value from the private source (`it--dev-settings`) only if `gh` auth is available.
If `gh` is not available or not authenticated, skip this strategy and note it in the report.
For each value retrieved, grep all non-excluded tracked files for that literal value. Flag any match as a finding.

#### Key naming — shared vs repo-specific credentials

When suggesting a key name for each finding, apply this heuristic **before** defaulting to `rise.<repo>.<purpose>`:

**Use `rise.shared.<purpose>` when the credential is org-wide** — the same value is used by every Rise project in development. Known shared credentials:

| Variable pattern | Suggested key | Rationale |
|-----------------|---------------|-----------|
| `TenantId`, `TenantName` | `rise.shared.entra-tenant-id` / `rise.shared.entra-tenant-name` | One Azure AD tenant for the whole org |
| Third-party dev API keys reused across repos (e.g. `CountryStateCityApi__ApiKey`, `GoogleMaps__ApiKey`) | `rise.shared.<vendor>-api-key` | Shared dev-tier key; single entry in `it--dev-settings` |

**Use `rise.<repo>.<purpose>` when the credential is repo-specific** — it differs per application or environment:

| Variable pattern | Example key | Rationale |
|-----------------|-------------|-----------|
| `ClientId`, `AppId`, `Audience` | `rise.<repo>.entra-client-id` | Each app has its own App Registration |
| `ConnectionStrings:*`, `*_CONNECTION_STRING` | `rise.<repo>.db-connection-string` | Connection strings include db name and server, which differ per repo |
| `ClientSecret`, `*_SECRET` | `rise.<repo>.entra-client-secret` | Per-app secret |

When in doubt, default to repo-specific. The developer can correct it in the Phase 5 review gate. Always note in the report if you are proposing a shared key, so the developer can verify with Guillaume that the key already exists in `it--dev-settings`.

### Phase 3 — Report and manifest draft

**Important**: Do not classify any finding as "INFO", "no action required", or "no action needed". Every variable found in a committed config file is a finding that gets a manifest entry — the manifest purpose is centralised rotation, not only security. Values like Entra TenantId, ClientId, and Audience belong in the manifest because they may need to be rotated and should not be hardcoded in every repo.

Group findings by detection method. Show summary counts first.

For each finding:
- File path and line number
- Detection method
- Redacted preview (first 4 chars + `***` — never the full value)
- Suggested `dev-settings.json` entry (key name, targetFile, targetVariable)
- If the suggested key uses `rise.shared.*`: flag it with a note — "Proposed as shared key — verify with Guillaume that this entry already exists in `it--dev-settings` before adding a duplicate."

**After presenting the report**, write `.claude/dev-settings.json` with all suggested manifest entries. This write is intentionally autonomous and ungated — the file is a draft manifest for human review with no runtime effect until the developer deliberately runs `/project-inject-dev-settings`. Gating this write would break the scan-then-correct workflow the developer expects.
- If `.claude/dev-settings.json` already exists, preserve existing entries and append new ones (do not duplicate entries with the same key).
- If it does not exist, create it.
- Use the manifest format from `_dev-settings-conventions.md`.
- Inform the developer: "`.claude/dev-settings.json` has been written with N entries. Review and correct the key names, targetFile, and targetVariable values before running `/project-inject-dev-settings`."

Conclude with a **"What to do"** section listing:
1. Keys to add to `it--dev-settings` (contact Guillaume)
2. Entries in `.claude/dev-settings.json` to review and correct (key names, targetFile, targetVariable)
3. Next: run `/project-migrate-dev-settings sanitize` to replace exposed values in committed files with placeholders and optionally purge git history

If no findings: report clean with scan summary (files scanned, strategies used). Do not write an empty manifest file.

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

- **REQ-2.4**: Replace the exposed value in the **committed source file** with a placeholder (for dotenv-style: `KEY=<injected>`; for JSON: `"<injected>"`). This file keeps its placeholder in git — it is the template. Do not add it to `.gitignore`.
- **REQ-2.5**: For Strategy B findings, the `targetFile` is the gitignored injection target (e.g. `appsettings.local.json`, `.env.local`) — not the committed source file. Add the `targetFile` to `.gitignore` if it is not already there. This is the file the injection skill writes real values into; the committed source file stays in git with placeholders.

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
