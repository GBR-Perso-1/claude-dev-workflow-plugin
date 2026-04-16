# Dev Workflow Plugin — Evolution Requirements

## Vision

Add a `/repo-archaeologist` skill that gives a developer instant business-level
understanding of any unfamiliar codebase. It performs a full deep scan — treating
code as the source of truth — and produces a structured report covering purpose,
key workflows, notable quirks, and documentation gaps. The primary use case is
onboarding onto a repo you haven't touched in months, or have never seen before.

## Epics

### Epic 1: repo-archaeologist Skill

**Goal**: Run `/repo-archaeologist` in any repo and receive a clear, code-grounded
understanding of what it does, how it works, and where the documentation falls short.

#### Requirements

- **REQ-1.1**: The skill performs a full deep scan of the repository — entry points,
  API surfaces, UI routes, internal services, utilities, and especially data models,
  which are treated as the most reliable representation of domain reality.

- **REQ-1.2**: Code is the authoritative source of truth. Documentation is read and
  cross-referenced, but never trusted over what the code actually does.

- **REQ-1.3**: The skill produces a Markdown report saved to
  `.claude/reports/archaeology-YYYY-MM-DD.md` (date = the day the skill is run).
  If the file already exists (same-day re-run), it is overwritten. The `.claude/reports/`
  directory is created automatically if it does not exist.

- **REQ-1.4**: The report is structured in this order:
  1. **Purpose** — what this repo is for, in plain business language
  2. **Key Workflows** — the main processes or user journeys the system supports
  3. **Notable Quirks** — unusual patterns, complex business rules, non-obvious
     decisions, or anything a new developer would trip over
  4. **Documentation Gaps** — a dedicated section listing all identified gaps
     (see REQ-1.5)

- **REQ-1.5**: The Documentation Gaps section captures all three gap types:
  - Code that has no corresponding documentation or explanation
  - Documentation that describes behaviour the code no longer implements
  - Business rules embedded in code with no explanation of *why*

- **REQ-1.6**: Once the report is saved, the skill surfaces a high-level summary
  directly in the conversation covering: purpose, key workflows, notable quirks,
  and total gap count — plus the path to the full report.

- **REQ-1.7**: The skill is implemented as a dedicated agent whose sole
  responsibility is producing this report. It does not modify any source files.

#### Decisions & Assumptions

- The skill targets the current working directory (the repo you are already in).
  Pointing it at an external path is out of scope for now.
- No severity tiers on documentation gaps in this iteration — all gaps are listed
  equally. Prioritisation can be added later.
- Re-run on the same day overwrites silently — no versioning or diff between runs
  in this iteration.

## Priorities

| Priority | Epic / Requirement | Rationale |
| -------- | ------------------ | --------- |
| 1        | REQ-1.1 — Deep scan incl. data models | Core value; data models are the most reliable signal |
| 2        | REQ-1.4 — Structured report | Determines usefulness of the output |
| 3        | REQ-1.5 — Documentation gaps section | Key differentiator vs. a basic README summary |
| 4        | REQ-1.6 — Conversation summary + path | UX polish; makes the skill feel responsive |
| 5        | REQ-1.3 — File output & auto-create dir | Persistence; needed for the report to be shareable |

## Out of Scope

- Pointing the skill at a repo path other than the current working directory
- Gap severity tiers / prioritisation
- Diff between successive runs on the same day
- Automated suggestions for fixing documentation gaps
