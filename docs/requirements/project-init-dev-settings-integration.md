# Claude Dev Workflow Plugin — Evolution Requirements

## Vision

Reduce the cognitive overhead of bootstrapping a project for AI-assisted development.
Today, a developer cloning a project must remember to run both `/project-init` and
`/project-project-inject-dev-settings` in sequence. This evolution makes `/project-init` the single
entry point for project readiness — it becomes aware of dev settings during bootstrap
and either helps create the manifest or offers to inject credentials, depending on
what it finds. `/project-project-inject-dev-settings` remains as a standalone skill for recurring
credential refreshes.

## Epics

### Epic 1: Dev Settings Awareness in /project-init

**Goal**: A developer bootstrapping a project on a new machine runs one command —
`/project-init` — and ends up with a fully configured, credential-injected environment,
without needing to know about `/project-project-inject-dev-settings` upfront.

#### Requirements

- **REQ-1.1**: When `/project-init` runs in Full Bootstrap mode and no
  `.claude/dev-settings.json` exists, it asks the user whether the project
  requires injected dev credentials before completing.

- **REQ-1.2**: If the user confirms, `/project-init` presents an interactive wizard
  that collects one or more manifest entries. Each entry requires: `key` (following the
  `<org>.<repo>.<purpose>` convention), `targetFile` (relative path), and
  `targetVariable` (variable name). The wizard allows adding multiple entries before
  saving.

- **REQ-1.3**: The wizard validates each entry before saving — all three fields must be
  present and non-empty. The user must correct invalid entries before the manifest is
  written.

- **REQ-1.4**: Once the wizard completes, `/project-init` saves the manifest to
  `.claude/dev-settings.json` and does not attempt injection. The summary directs the
  user to run `/project-project-inject-dev-settings` once the keys have been added to the private source.

- **REQ-1.5**: When `/project-init` runs in Full Bootstrap mode and a
  `.claude/dev-settings.json` already exists (e.g., cloned from the repo), it asks the
  user whether to inject credentials now before completing.

- **REQ-1.6**: If the user confirms, `/project-init` runs the full injection flow
  (equivalent to `/project-project-inject-dev-settings`) as its final step, including all preflight
  checks, gitignore guards, and injection logic.

- **REQ-1.7**: If the user declines injection (REQ-1.5), the summary reminds them
  to run `/project-project-inject-dev-settings` to hydrate the local environment.

- **REQ-1.8**: When `/project-init` runs in **Refresh Rules Only** mode (`.claude/CLAUDE.md`
  already exists), dev settings are not mentioned and no wizard or injection is triggered.

- **REQ-1.9**: `/project-init`'s final summary includes a dev settings section when
  relevant — reporting whether the manifest was created, whether injection ran, and
  whether any gitignore additions were made.

#### Decisions & Assumptions

- Injection during bootstrap is opt-in, not automatic — the user confirms before
  credentials are fetched.
- The manifest wizard does not validate key names against the private source. If a key
  doesn't exist there, injection will fail with a clear message from the existing
  project-inject-dev-settings logic.
- `/project-project-inject-dev-settings` is not modified by this epic — it remains a standalone, fully
  functional skill.
- The injection flow called by `/project-init` in REQ-1.6 must replicate all guardrails
  from `/project-project-inject-dev-settings` (no credential values in output, no staging of target
  files).

## Priorities

| Priority | Epic / Requirement | Rationale |
|----------|--------------------|-----------|
| 1 | REQ-1.5, REQ-1.6, REQ-1.7 | Highest-value case: cloning an existing project that already has a manifest. Covers the most common "one command to be ready" scenario. |
| 2 | REQ-1.1, REQ-1.2, REQ-1.3, REQ-1.4 | New project bootstrap wizard. Valuable but less frequent — new projects are rarer than clones. |
| 3 | REQ-1.8, REQ-1.9 | Guardrails and summary polish — low effort, high clarity. |

## Out of Scope

- Merging `/project-project-inject-dev-settings` into `/project-init` as a single skill — injection
  remains a standalone command for recurring use.
- Automatic injection without user confirmation during bootstrap.
- `/project-init` listing or suggesting keys from the private source.
- Any changes to the Refresh Rules Only mode of `/project-init`.
- Admin functionality for adding keys to `Rise-4/rise-dev-settings`.
