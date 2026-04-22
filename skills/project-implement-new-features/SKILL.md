---
name: project-implement-new-features
description: Implement a feature from a requirements document. Orchestrates architect, developer, tester, and reviewer agents in a feedback loop.
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Input

The user provides a requirement — either a path to a requirements document (from `/brainstorm`) or an inline description. If a path is given, read the file first.

## Instructions

### Phase 1 — Architecture (Architect Agent)

1. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/architect.md`.
   - Pass it the full requirement text and instruct it to produce a concrete implementation plan.
   - The plan must include: affected files, new files to create, data model changes, API changes, UI changes, and a step-by-step implementation order.
2. Present the architect's plan to the user.
3. Use `AskUserQuestion` with options:
   - "Looks good — proceed to implementation (Recommended)"
   - "I want to refine the plan"
   - "Start over with different requirements"
4. If the user wants refinements, relay their feedback to the architect agent (via `SendMessage`) and repeat steps 2–3 until approved.

### Phase 2 — Implementation (Developer Agent)

5. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/developer.md`.
   - Pass it the **approved architecture plan** and the **original requirements**.
   - Instruct it to implement all changes described in the plan.
6. Wait for the developer agent to complete. Collect its summary of changes made.

### Phase 3 — Testing (Test Agent)

7. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/test-writer.md`.
   - Pass it: (1) the **original requirements**, (2) the **Test Strategy section** from the architect's plan.
   - The test-writer derives test scenarios from the requirements — not from what the developer coded.
8. Evaluate the test report:
   - **All pass** → proceed to Phase 4.
   - **Failures due to source bugs** → pass the failing test details and bug descriptions back to the **developer agent** (via `SendMessage`) and instruct it to fix the bugs. Then re-run Phase 3 (spawn a fresh test-writer agent).
   - **Maximum 3 iterations** of the dev↔test loop. If tests still fail after 3 rounds, present the situation to the user via `AskUserQuestion`:
     - "Continue iterating"
     - "Skip failing tests and proceed to review"
     - "Stop here — I'll fix manually"

### Phase 4 — Review (Quality + Design Agents)

9. Spawn **three agents in parallel**:
   - **Code-quality reviewer**: the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/reviewer-quality.md` — checks linting, style, and rule compliance.
   - **Design reviewer**: the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/reviewer-design.md` — checks requirement coverage, domain boundaries, abstraction quality, and consistency.
   - **Performance reviewer**: the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/reviewer-perf.md` — checks EF queries, unbounded loads, N+1 patterns, caching gaps, and frontend perf.
10. Collect all three reports and present them to the user.

11. Evaluate findings:
    - **No violations or warnings** → proceed to Phase 5.
    - **Implementation errors only** (code quality issues, bugs, style) → pass findings to the **developer agent** (via `SendMessage`) for corrections. Then re-run Phase 3 (testing) and Phase 4 (review).
    - **Design errors** (wrong abstraction, domain boundary violation, requirement mismatch, missing functionality) → pass findings back to the **architect agent** (via `SendMessage`) to revise the plan. Then re-run from Phase 2.
    - **Maximum 3 iterations** of the full review loop. If issues persist, present to the user via `AskUserQuestion`:
      - "Continue iterating"
      - "Accept current state — I'll handle remaining issues"
      - "Stop and roll back"

### Phase 5 — Done

12. Present a final summary to the user:
    - What was implemented (files created/modified)
    - Test results (pass/fail counts)
    - Review outcome (clean / accepted with notes)
13. Use `AskUserQuestion` with options:
    - "Commit these changes (Recommended)"
    - "I want to review the changes manually"

## Orchestration Rules

- **Always use `SendMessage`** to continue an existing agent rather than spawning a new one (except for the test-writer, which should be spawned fresh each iteration since it re-analyses changes from scratch).
- **Never skip phases** — every implementation must go through architect → dev → test → review.
- **Never apply fixes yourself** — always delegate to the appropriate agent.
- **Track iteration counts** and enforce the maximum of 3 per loop to avoid infinite cycles.
- When routing errors back to agents, include the **specific findings** and **file/line references** so the agent has full context.
- Use British English throughout.

## Conversation Style

- Be a **project manager** — coordinate, summarise progress, and keep things moving.
- After each phase, give the user a brief status update before proceeding.
- If an agent produces unexpected output, flag it to the user rather than guessing.
