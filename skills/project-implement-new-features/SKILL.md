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
   - Instruct it to implement all changes described in the plan, then **run the test suite inline** before returning (e.g. `dotnet test` / `npm test`). It should report which tests passed, failed, or were skipped — this is a quick smoke check, not a full test pass.
6. Wait for the developer agent to complete. Collect its summary of changes made and the inline test results.

### Phase 3 — Testing (Test Agent)

7. If the developer's inline test run from Phase 2 showed failures unrelated to missing test coverage (i.e. source bugs), pass the failures back to the **developer agent** via `SendMessage` to fix them before proceeding. Skip directly to Phase 4 if the inline run was clean.

8. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/test-writer.md`.
   - Pass it: (1) the **original requirements**, (2) the **Test Strategy section** from the architect's plan.
   - The test-writer derives test scenarios from the requirements — not from what the developer coded.
9. Evaluate the test report:
   - **All pass** → proceed to Phase 4.
   - **Failures due to source bugs** → pass the failing test details back to the **developer agent** (via `SendMessage`) to fix. Then re-run Phase 3 (spawn a fresh test-writer agent).
   - **Maximum 2 iterations** of the dev↔test loop. If tests still fail after 2 rounds, present via `AskUserQuestion`:
     - "Continue iterating"
     - "Skip failing tests and proceed to review"
     - "Stop here — I'll fix manually"

### Phase 4 — Review (Quality + Design Agents)

9. **Stack detection** — before spawning reviewers, run these checks in parallel:
   ```bash
   # EF Core present?
   grep -rl "EntityFrameworkCore" --include="*.csproj" . 2>/dev/null | head -1
   # Vue/Quasar frontend present?
   node -e "const p=require('./app/package.json'); console.log(p.dependencies?.vue||p.dependencies?.quasar||'')" 2>/dev/null
   ```
   - If either check returns a non-empty result → **perf review applies**, spawn all three reviewers.
   - If both return empty → **perf review skipped**, spawn quality + design only.

10. Spawn reviewers in parallel (two or three depending on stack detection):
    - **Code-quality reviewer**: `${CLAUDE_PLUGIN_ROOT}/agents/reviewer-quality.md` — linting, style, rule compliance.
    - **Design reviewer**: `${CLAUDE_PLUGIN_ROOT}/agents/reviewer-design.md` — requirement coverage, domain boundaries, abstraction quality, consistency.
    - **Performance reviewer** *(only when stack detected)*: `${CLAUDE_PLUGIN_ROOT}/agents/reviewer-perf.md` — EF queries, unbounded loads, N+1 patterns, caching gaps, frontend perf.

11. Collect all reports. Record which reviewers **passed** and which **flagged issues**. Present findings to the user.

12. Evaluate findings:
    - **No violations or warnings** → proceed to Phase 5.
    - **Implementation errors only** (code quality issues, bugs, style) → pass findings to the **developer agent** (via `SendMessage`) for corrections. Re-run Phase 3 (testing), then **re-run only the reviewers that previously flagged issues** — skip reviewers that already passed.
    - **Design errors** (wrong abstraction, domain boundary violation, requirement mismatch, missing functionality) → pass findings back to the **architect agent** (via `SendMessage`) to revise the plan. Re-run from Phase 2 with all reviewers reset.
    - **Maximum 2 iterations** of the full review loop. If issues persist, present via `AskUserQuestion`:
      - "Continue iterating"
      - "Accept current state — I'll handle remaining issues"
      - "Stop and roll back"

### Phase 5 — Done

13. Present a final summary to the user:
    - What was implemented (files created/modified)
    - Test results (pass/fail counts)
    - Review outcome (clean / accepted with notes)
14. Use `AskUserQuestion` with options:
    - "Commit these changes (Recommended)"
    - "I want to review the changes manually"

## Orchestration Rules

- **Always use `SendMessage`** to continue an existing agent rather than spawning a new one (except for the test-writer, which should be spawned fresh each iteration since it re-analyses changes from scratch).
- **Never skip phases** — every implementation must go through architect → dev → test → review.
- **Never apply fixes yourself** — always delegate to the appropriate agent.
- **Track iteration counts** and enforce the maximum of **2 per loop** to avoid infinite cycles.
- **Only re-run reviewers that flagged issues** in iteration 2 — do not re-spawn reviewers that already passed.
- When routing errors back to agents, include the **specific findings** and **file/line references** so the agent has full context.
- Use British English throughout.

## Conversation Style

- Be a **project manager** — coordinate, summarise progress, and keep things moving.
- After each phase, give the user a brief status update before proceeding.
- If an agent produces unexpected output, flag it to the user rather than guessing.
