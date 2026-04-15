---
name: plugin-implement
description: Implement new or updated plugin artefacts (skills, agents, hooks, rules) with full awareness of the existing plugin ecosystem.
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Input

The requirement comes from `$ARGUMENTS`. If `$ARGUMENTS` is empty, use `AskUserQuestion` to request a description of what needs to be built or changed.

## Instructions

### Phase 1 — Ecosystem Survey & Design (Plugin Architect Agent)

1. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/plugin-architect.md`.
   - Pass it: (a) the full requirement text, (b) instruction to perform the ecosystem survey first and embed the Ecosystem Snapshot in its plan output.
2. Present the architect's plan to the user (ecosystem snapshot + conflict analysis + artefact list).
3. Use `AskUserQuestion` with options:
   - "Looks good — proceed to implementation (Recommended)"
   - "I want to refine the plan"
   - "Start over with different requirements"
4. If refinement requested, relay feedback via `SendMessage` to the same architect instance and repeat steps 2–3 until approved.

### Phase 2 — Implementation (Developer Agent)

5. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/developer.md`.
   - Pass it: the approved plan, the original requirement, and this explicit instruction: **"You are implementing Markdown artefact files for a Claude Code plugin. There is no compilation step — do not run `dotnet build`, `npm run type-check`, or any build command. Instead, verify that every file listed in the plan has been created or modified and that all Markdown files are well-formed."**
6. Wait for the developer's implementation report listing all files created/modified.

### Phase 3 — Validation (Plugin Reviewer Agent)

7. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/plugin-reviewer.md`.
   - Pass it: (a) the original requirement, (b) the architect's approved plan, (c) the list of changed files from the developer's report.
   - Instruct it to validate all changed files against plugin conventions.
8. Collect the review report.

### Phase 4 — Feedback Loop

9. Evaluate findings:
   - **No violations or warnings** → proceed to Phase 5.
   - **Convention errors** (SKILL format, frontmatter, bad agent refs) → pass findings to the developer agent via `SendMessage` for corrections. Continue the existing `plugin-reviewer` agent via `SendMessage` and instruct it to re-validate the corrected files.
   - **Design errors** (missing artefacts, wrong structure, ecosystem conflicts not resolved) → pass findings to the architect agent via `SendMessage` to revise the plan. Re-run from Phase 2.
   - **Maximum 3 iterations**. If issues persist after 3 rounds, use `AskUserQuestion`:
     - "Continue iterating"
     - "Accept current state — I'll handle remaining issues manually"
     - "Stop here"

### Phase 5 — Manifest Update & Done

10. If the plan indicates `plugin.json` requires a version bump, and the developer has not already done it, pass that instruction to the developer via `SendMessage`.
11. Present a final summary:
    - Artefacts created/modified (file list)
    - Review outcome (clean / accepted with notes)
12. Use `AskUserQuestion` with options:
    - "Commit these changes (Recommended)"
    - "I want to review the changes manually first"

## Orchestration Rules

- Always use `SendMessage` to continue an existing agent rather than spawning a new one for the same role.
- Never skip phases.
- Never apply fixes yourself — always delegate to the appropriate agent.
- Track iteration counts; enforce maximum of 3 per loop.
- When routing errors back to agents, include specific findings and file references.
- Use British English throughout.

## Conversation Style

- Be a **project manager** — coordinate, summarise progress, and keep things moving.
- After each phase, give the user a brief status update before proceeding.
- If an agent produces unexpected output, flag it rather than guessing.
