# Claude Dev Workflow Plugin — New Skill Requirements

## Vision

Add a `project-migrate-dev-settings` skill that helps developers discover secrets
accidentally committed to a repository and migrate them into the dev-settings pattern
(`it--dev-settings` + `.claude/dev-settings.json`). The skill operates in two
modes: a safe scan-and-report mode (default) and an autonomous sanitise mode (invoked
with the `sanitize` argument) that rewrites both the working tree and git history,
with a single confirmation gate before the destructive steps.

## Epics

### Epic 1: Scan & Report (Default Mode)

**Goal**: Give the developer a clear picture of what secrets are exposed in the repo,
what the dev-settings manifest should contain, and what keys need to be added to the
private source — without touching anything.

#### Requirements

- **REQ-1.1**: The skill scans the repo using three complementary detection strategies:
  - **Pattern matching** — identify high-entropy strings, known API key formats (e.g.
    Bearer tokens, connection strings with passwords, private key blocks), and common
    secret variable names assigned to non-empty string literals.
  - **Committed env/config files** — flag files that match local-config patterns
    (`.env`, `.env.*`, `appsettings.*.json`, `secrets.json`, `local.settings.json`,
    etc.) that are tracked by git but should not be committed.
  - **Manifest cross-reference** — if `.claude/dev-settings.json` already exists, locate
    any file in the repo that contains the actual value of a key listed in the manifest.

- **REQ-1.2**: The scan excludes paths that are already gitignored, binary files, and
  known safe directories (`node_modules/`, `vendor/`, `.git/`, build output folders).

- **REQ-1.3**: For each finding, the report shows: the file path and line number, the
  detection method that triggered it, a redacted preview of the value (never the full
  value), and a suggested `dev-settings.json` entry (key name following
  `<org>.<repo>.<purpose>`, target file, target variable).

- **REQ-1.4**: The report groups findings by detection method and presents a summary
  count before the detail.

- **REQ-1.5**: The report concludes with a "What to do" section listing the keys that
  would need to be added to `it--dev-settings` and directing the developer to
  run `/project-migrate-dev-settings sanitize` to act on the findings.

- **REQ-1.6**: If no secrets are found, the skill reports clean with a brief summary of
  what was scanned.

---

### Epic 2: Sanitise Mode (`sanitize` argument)

**Goal**: Autonomously clean the repo — working tree and git history — with a single
human confirmation gate before the irreversible steps.

#### Requirements

**Phase A — Per-secret review (before any changes)**

- **REQ-2.1**: The skill runs the same full scan as Epic 1. Before making any changes,
  it presents each finding one at a time and asks the developer to confirm or edit the
  proposed `dev-settings.json` entry (key name, target file, target variable) via
  `AskUserQuestion`.

- **REQ-2.2**: For each finding, the developer can also choose to **skip** it (exclude
  from sanitisation, e.g. a false positive). Skipped findings are listed in the final
  summary but never touched.

- **REQ-2.3**: If all findings are skipped, the skill halts with a message and makes no
  changes.

**Phase B — Working tree changes (autonomous after review)**

- **REQ-2.4**: For each approved finding, the skill replaces the exposed secret value in
  the source file with a reference to the target variable (e.g. an environment variable
  lookup), preserving the surrounding file structure.

- **REQ-2.5**: For each committed env/config file finding, the skill removes the secret
  value from the file (replaces the line with a placeholder comment or blank value) and
  adds the file to `.gitignore`.

- **REQ-2.6**: The skill creates or updates `.claude/dev-settings.json` with all
  approved entries. Existing entries in the manifest are preserved; new entries are
  appended.

- **REQ-2.7**: All target files referenced in new manifest entries are checked against
  `.gitignore` and added if missing.

- **REQ-2.8**: The skill does **not** attempt to write values to `it--dev-settings`
  at any point.

**Phase C — Confirmation gate**

- **REQ-2.9**: Before touching git history, the skill pauses and presents a summary of:
  the commits that contain the secrets (count and earliest commit), the files and
  variable names affected, and a clear warning that the history rewrite will require a
  force push and will invalidate any collaborators' local history.

- **REQ-2.10**: The developer must explicitly confirm via `AskUserQuestion` before the
  history rewrite begins. "Cancel" at this gate aborts history rewrite but keeps the
  working tree changes already made.

**Phase D — History purge (after confirmation)**

- **REQ-2.11**: The skill purges all approved secret values from the full git history
  across all branches that are checked out locally.

- **REQ-2.12**: After the history rewrite, the skill force-pushes the affected branches
  to the remote. The developer must confirm the branch name and remote before the push.

- **REQ-2.13**: If the history rewrite or force push fails, the skill surfaces the error
  and halts without retrying — the developer must resolve manually.

**Phase E — Private source handoff**

- **REQ-2.14**: After history purge, the skill displays the list of keys that must be
  added to `it--dev-settings` along with a reminder of the key naming convention
  and instructions for contacting the platform admin (Guillaume).

- **REQ-2.15**: The skill waits for the developer to confirm they have added the keys to
  the private source before offering to run `/project-inject-dev-settings`.

- **REQ-2.16**: Once the developer confirms, the skill offers to run
  `/project-inject-dev-settings` to immediately hydrate the local environment.

**Phase F — Summary**

- **REQ-2.17**: The skill prints a final summary covering: findings processed vs skipped,
  files modified in the working tree, git history rewritten (yes/no), branches
  force-pushed, and whether injection was run.

- **REQ-2.18**: The summary includes a **rotate secrets** warning for every secret that
  was found in git history — the developer must rotate those credentials because they
  remain accessible in any clone made before the rewrite.

---

## Priorities

| Priority | Epic / Requirement | Rationale |
|----------|--------------------|-----------|
| 1 | REQ-1.1 – REQ-1.6 | Scan & report is safe and immediately useful. Can be used independently before committing to sanitise. |
| 2 | REQ-2.1 – REQ-2.3 | Per-secret review must come first — it gates everything else. |
| 3 | REQ-2.4 – REQ-2.8 | Working tree changes are the core value; lower risk than history ops. |
| 4 | REQ-2.9 – REQ-2.13 | History purge — high value but destructive; depends on working tree phase. |
| 5 | REQ-2.14 – REQ-2.18 | Handoff + summary — polish, but important for the full workflow to close cleanly. |

## Out of Scope

- Automatically writing secret values to `it--dev-settings` — admin-controlled only.
- Purging secrets from branches not checked out locally.
- Handling secrets in git submodules.
- Notifying collaborators after a history rewrite.
- Detecting secrets in CI/CD pipeline files or GitHub Actions secrets (separate concern).
