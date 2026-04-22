---
name: project-implement-fix
description: Lightweight implementation skill for small fixes and changes. Interprets the user's request, delegates to the developer, runs a short test loop, and stops — no architect agent, no reviewers, no commit.
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Input

The user provides a brief description of what they want fixed or changed — inline text or a short note. No requirements document needed.

## Instructions

### Phase 0 — Interpret the request

1. Read the user's request and restate it as a clear, concise implementation brief:
   - What needs to change and where (file/component/layer if known)
   - What the expected outcome is
   - Any constraints or edge cases the user mentioned
2. Present the brief to the user and confirm before proceeding. Keep this step short — one paragraph max.

### Phase 1 — Implementation (Developer Agent)

3. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/developer.md`.
   - Pass it the implementation brief from Phase 0.
   - Instruct it to implement only what is described — no scope creep.
4. Wait for the developer agent to complete. Collect its summary of changes made.

### Phase 2 — Testing (Test Agent)

5. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/test-writer.md`.
   - Pass it: (1) the implementation brief, (2) the developer's summary of changes.
6. Evaluate the test report:
   - **All pass** → proceed to Phase 3.
   - **Failures due to source bugs** → pass the failing test details back to the **developer agent** (via `SendMessage`) and instruct it to fix the bugs. Spawn a fresh test-writer agent and re-run Phase 2.
   - **Maximum 2 iterations** of the dev↔test loop. If tests still fail after 2 rounds, present the situation to the user via `AskUserQuestion`:
     - "Continue iterating"
     - "Stop here — I'll fix manually"

### Phase 3 — Done

7. Present a final summary to the user:
   - What was changed (files modified)
   - Test results (pass/fail counts)
   - A reminder to review the diff before committing

**Never commit.** Always end here and leave the user to review and commit.

## Orchestration Rules

- **Always use `SendMessage`** to continue an existing agent rather than spawning a new one (except for the test-writer, which should be spawned fresh each iteration).
- **No architect, no reviewers** — this skill is intentionally lightweight.
- **Never apply fixes yourself** — always delegate to the developer agent.
- **Track iteration counts** and enforce the maximum of 2 per loop.
- Use British English throughout.

## Conversation Style

- Be brief and direct — this is a quick-fix flow, not a project.
- Phase 0 confirmation should be a single short paragraph, not a formal document.
- After the test phase, give a one-line status before wrapping up.
