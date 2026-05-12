# dev-workflow Plugin — Evolution Requirements: project-decide skill

## Vision

Add a new `project-decide` skill that acts as a decision layer between the
investigation/debug skills and the implementation skills. It reads findings
already in the conversation, generates plausible solution options autonomously,
evaluates each with pros and cons, and commits to a recommended option — with
long-term maintainability and architectural purity as the primary evaluation
lens. It never modifies code or infrastructure.

---

## Epics

### Epic 1: project-decide skill

**Goal**: Give the user a structured, opinionated options report after an
investigation or debug session, so they arrive at the implementation step with
a clear preferred direction — not just raw findings.

#### Requirements

- **REQ-1.1**: The skill accepts an investigation report or debug report
  already present in the conversation as its primary input. It must not require
  the user to re-run a scan or re-paste findings; the conversation context is
  sufficient.

- **REQ-1.2**: `$ARGUMENTS` may optionally carry a short user-supplied framing
  (e.g. "focus on the auth layer", "we can't touch the DB schema"). If absent,
  the skill infers scope from the report in context.

- **REQ-1.3**: The skill restates the core problem to be solved — derived from
  the investigation findings — and asks the user to confirm or correct it before
  generating options. This prevents the skill from solving the wrong problem.

- **REQ-1.4**: The skill generates 2–4 distinct solution options autonomously
  by reasoning over the findings. Options must be meaningfully different
  directions, not variations of the same approach. At least one option must
  represent the "smallest safe change" and at least one must represent the
  "most structurally correct" solution.

- **REQ-1.5**: For each option the skill produces:
  - A plain-language description of the approach (what would change, at the
    conceptual level — no code snippets or file edits)
  - A bullet list of pros
  - A bullet list of cons
  - An effort signal: Low / Medium / High (relative to the other options in
    this report)

- **REQ-1.6**: The skill selects one **Recommended Option** and states it
  clearly, with a short rationale grounded in long-term maintainability and
  architectural purity. If a different option would be preferred under specific
  conditions (e.g. "if a quick hotfix is needed before the next release"),
  that conditional preference is stated alongside the main recommendation.

- **REQ-1.7**: The skill is strictly read-only. It must not modify source
  files, config, Azure resources, or any project artefact.

- **REQ-1.8**: The final report ends with a handoff pointer directing the user
  to `/project-implement-fix` (for contained fixes) or
  `/project-implement-new-features` (for larger changes), so the transition
  from decision to action is explicit.

#### Decisions & Assumptions

- Option count capped at 4 to keep the report actionable. If more than 4
  genuinely distinct directions exist, the skill groups the closest ones and
  notes the grouping.
- "Architectural purity" is assessed relative to the conventions observed in the
  investigation report, not against an external standard — the skill must not
  invent principles that aren't evidenced in the codebase.
- The skill does not re-scan the codebase. If the investigation report contains
  insufficient evidence to reason about a particular option, the skill flags
  this as an uncertainty rather than scanning independently.
- Skill name: `project-decide` — consistent with the `project-*` prefix
  convention for skills that operate at the project-analysis level.

---

## Priorities

| Priority | Epic / Requirement            | Rationale                                                    |
| -------- | ----------------------------- | ------------------------------------------------------------ |
| 1        | REQ-1.3 (confirm the problem) | Solving the wrong problem wastes the user's time             |
| 2        | REQ-1.4 (generate options)    | Core value of the skill                                      |
| 3        | REQ-1.6 (recommendation)      | The "tell me the best one" ask is the main differentiator    |
| 4        | REQ-1.5 (pros/cons/effort)    | Supports the recommendation; prevents black-box advice       |
| 5        | REQ-1.8 (handoff pointer)     | Completes the investigate → decide → implement chain         |

## Out of Scope

- Re-running a codebase scan or spawning investigation agents (REQ-1.7)
- Asking the user to supply their own options before evaluation
- Generating implementation plans, code snippets, or file-level changes
- Producing a requirements document (that's `/project-brainstorm`)
- Saving the options report to disk — it lives in the conversation
