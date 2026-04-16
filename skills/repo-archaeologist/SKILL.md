---
name: repo-archaeologist
description: Perform a full deep scan of the current repository and produce a structured archaeology report covering business purpose, key workflows, notable quirks, and documentation gaps. Saves the report to .claude/reports/archaeology-YYYY-MM-DD.md.
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Constraints

- **No arguments required** — this skill always operates on the current working directory.
- **No source file modifications** — neither this skill nor the agent it spawns may modify source files.

## Instructions

### Phase 1 — Run the Archaeology Agent

1. Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/repo-archaeologist.md`.
   - Pass no additional arguments — the agent operates on the current working directory.
   - Instruct it to complete all five phases and return its structured summary.
2. Wait for the agent to return its summary. Do not proceed until the summary is received.

### Phase 2 — Surface Summary to User

3. Present the agent's summary directly in the conversation using this structure:

```markdown
## Repository Archaeology — Summary

### Purpose
<agent's Purpose paragraph>

### Key Workflows
<agent's Key Workflows list>

### Notable Quirks
<agent's Notable Quirks list>

### Documentation Gaps
Total: N gaps (Type A: N — undocumented code | Type B: N — stale docs | Type C: N — unexplained rules)

---
Full report saved to: <report path>
```

4. No confirmation gate is needed — the skill is read-only and produces no destructive changes.

## Conversation Style

- Present the summary cleanly — no preamble, no meta-commentary about the scan process.
- Use British English throughout.
