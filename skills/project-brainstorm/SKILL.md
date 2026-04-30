---
name: project-brainstorm
description: Brainstorm and write requirements for app evolution. Explore the codebase for context but never propose concrete implementation.
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Constraints

- **No code changes** — this skill is for ideation and requirement writing only.
- **No implementation details** — never propose specific technical solutions, architecture patterns, file changes, or code snippets.
- **Codebase as context only** — read the codebase to understand what exists today, not to plan how to change it.
- Output is always a **requirements document**, not a technical spec.

## Instructions

### Phase 0 — Quick scan

Using only Glob, Grep, and Read tools — without spawning any agents — perform a lightweight self-scan to build an initial "State of the App" snapshot.

1. Read `.claude/CLAUDE.md` if present — for project identity, stack, and any stated architectural guardrails.
2. Read `.claude/project-context.md` if present — for business context, team, and user base.
3. Use `Glob` with patterns `README.md`, `package.json`, `*.sln`, `*.csproj` to locate project metadata files; read the most informative one.
4. Use `Glob` to identify the top-level directory structure (e.g. `api/`, `app/`, `infra/`, `src/`) — note major areas.
5. Identify one or two key entry-point files (e.g. `Program.cs`, `main.ts`, router definition, or top-level page list) and read them.
6. Produce a **Quick Context Summary** in this format:

```markdown
## Quick Context Summary

### Project identity
<name, purpose, and stack — derived from CLAUDE.md / project-context.md / metadata files>

### Major areas
<bullet list of top-level directories or layers and their stated purpose>

### Key entry points found
<bullet list: file paths and what they reveal about the application's structure>

### Gaps
<what could not be determined without a deeper scan — e.g. domain model detail, full feature list, internal service responsibilities>
```

Present the Quick Context Summary to the user, then ask via `AskUserQuestion`:

- **Question**: "Quick scan complete. Is this enough context to start brainstorming, or do you want a deeper codebase overview first?"
- **Options**:
  - `This is enough — let's start brainstorming`
  - `Go deeper — run full codebase exploration first`

If the user selects **"This is enough"**: skip Phase 1 entirely. Proceed directly to Phase 2 (Brainstorming), starting at step 4 (the `AskUserQuestion` about which area to evolve). Use the Quick Context Summary as the "State of the App" context for the brainstorming conversation — do not re-run context gathering.

If the user selects **"Go deeper"**: proceed to Phase 1 below, which will spawn the codebase-explorer agent.

---

### Phase 1 — Context Gathering

1. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/codebase-explorer.md`.
2. Present the agent's output as a **"State of the App"** summary to the user — what exists, what the app currently does, who it serves.

### Phase 2 — Brainstorming

4. Ask the user what area(s) they want to evolve. Use `AskUserQuestion` with options like:
   - "Expand an existing feature"
   - "Add a completely new capability"
   - "Rethink the user experience"
   - "Add integrations / data sources"
   - "Other (describe)"
5. For each idea the user brings up:
   - **Explore** — ask open-ended questions to flesh out the idea (who benefits, what problem it solves, what would success look like)
   - **Challenge** — raise edge cases, user experience concerns, or scope questions — but stay at the _what_, never the _how_
   - **Resolve** — every question raised must be answered before moving on, even if the answer is approximate (e.g. "we'll refine this later" or "good enough for now"). Do not leave open questions dangling.
   - **Capture** — summarise the idea as a short requirement statement
   - **Split** — if the idea covers multiple independent concerns, propose splitting it into separate epics rather than letting scope grow unchecked.

### Phase 3 — Requirements Document

6. Once the user signals they're done brainstorming, compile everything into a structured requirements document. Use this format:

```markdown
# [App Name] — Evolution Requirements

## Vision

One paragraph describing the overall direction.

## Epics

### Epic 1: [Name]

**Goal**: What this achieves for users.

#### Requirements

- **REQ-1.1**: [Requirement statement — user-facing, testable]
- **REQ-1.2**: ...

#### Decisions & Assumptions

- [Any fuzzy or deferred answers acknowledged during brainstorming, e.g. "Exact threshold TBD — rough target is X"]

### Epic 2: [Name]

...

## Priorities

| Priority | Epic / Requirement | Rationale |
| -------- | ------------------ | --------- |
| 1        | ...                | ...       |

## Out of Scope

- Items explicitly deferred or rejected during brainstorming.
```

7. Present the draft to the user via `AskUserQuestion` with options:
   - "Looks good — save it"
   - "I want to revise some requirements"
   - "Let's brainstorm more before finalising"
   - "Split into separate documents — this covers too much"
   - "Other"

8. When approved, save the document to `docs/requirements/` with a descriptive filename (e.g. `docs/requirements/evolution-v2-pricing-overhaul.md`).

## Conversation Style

- Be a **thinking partner**, not a note-taker — push back, ask "why", suggest angles the user hasn't considered.
- Keep the energy collaborative and forward-looking.
- When the user drifts into implementation ("we could use a queue for that"), gently redirect: _"That's an implementation detail — for now, what should the user experience be?"_
- Use British English throughout.
