---
name: project-debug
description: Debug an issue by exploring the codebase and optionally querying Azure. Reports findings only — never modifies code or infrastructure.
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Constraints

- **Read-only.** This skill and every agent it spawns must never modify source files, configuration, or Azure resources.
- **No fixes.** Report findings; the user decides how to act on them.

## Input

`$ARGUMENTS` — a free-text description of the issue. If empty, ask the user to describe the bug or symptom before proceeding.

---

### Phase 0 — Understand the issue

1. Read `$ARGUMENTS`. If blank, stop and ask:
   Use `AskUserQuestion` to collect the issue description:
   - Question: "What issue or symptom would you like investigated? Include any error messages, affected component names, or reproduction steps you already have."
   - Option: `I'll describe it` (user provides free-text via the automatic Other input)

2. Restate the issue as a short investigation brief:
   - **Symptom** — what the user is seeing
   - **Suspected area** — component, layer, or service (if mentioned)
   - **Azure involved?** — does the description mention infrastructure, auth, Key Vault, app registrations, Azure AD, Entra, MSAL, tenant, subscription, Static Web App, App Service, managed identity, deployments, or cloud resources?

3. Present the brief and ask the user to confirm or correct it using `AskUserQuestion`:
   - `Looks right — start investigating`
   - `Let me clarify` (user provides correction)

---

### Phase 1 — Parallel investigation

Launch **both agents simultaneously** when Azure is relevant. Otherwise launch only the code agent.

### 1a — Code investigation (always)

Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/repo-archaeologist.md`.

Prompt it with:

```
IMPORTANT: You are being used for targeted bug investigation, not a repository overview. Override your standard output format entirely — do NOT write any file to .claude/reports/ and do NOT produce an archaeology report. Instead, produce only the Bug Location Report format specified below.

You are investigating a reported bug. Your goal is NOT to fix it — only to understand and locate it.

Issue brief:
<paste the brief from Phase 0>

Investigation tasks:
1. Identify the files, classes, and functions most likely involved in the symptom.
2. Trace the relevant data flow or call chain end-to-end.
3. Look for obvious defects: null paths, missing guards, incorrect logic, wrong assumptions, mismatched types, off-by-one errors, race conditions, misconfigured values.
4. Note any related tests (passing or failing) that are relevant.
5. Flag any code patterns that are surprising or that deviate from the surrounding conventions.

Output a structured Bug Location Report (see format below). Do NOT suggest fixes.

---

## Bug Location Report

### Suspected Root Cause
One paragraph. What you think is actually wrong and why.

### Evidence
Bullet list. For each piece of evidence: file path + line range, what it shows, and why it is relevant to the bug.

### Data / Call Flow
Step-by-step trace from entry point to the point of failure (file:line for each step where possible).

### Related Tests
List any test files that cover the affected area. Note if they are passing, failing, or missing.

### Uncertainties
What you could not determine from static analysis alone (e.g. runtime values, external state, environment-specific behaviour).
```

### 1b — Azure investigation (only when Azure is relevant)

Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/azure-investigator.md`.

Prompt it with:

```
You are investigating an Azure infrastructure issue related to a reported bug.
Your goal is NOT to fix anything — only to find and report relevant state.

Issue brief:
<paste the brief from Phase 0>

Begin Phase 0 only: run your detection flow and auth check, then stop and return your findings. Do not investigate yet.
```

Wait for the agent to return its Phase 0 output (active Azure context or "not logged in" state). Proceed to Phase 2.

---

### Phase 2 — Azure context confirmation gate

Receive the Phase 0 output from the azure-investigator agent.

If the agent reported it is **NOT logged in**: inform the user and skip Azure investigation. Include a note in the final report.

If the agent reported an **active context**: present it to the user via `AskUserQuestion`:

The Azure investigation agent is connected to tenant <tenantId>, subscription <name> (<subscriptionId>), as user <user>. Is this the correct Azure environment for this investigation?

Options:
- `Yes — correct environment, continue`
- `No — skip Azure investigation`

If the user confirms: send a `SendMessage` to the existing azure-investigator agent to proceed with Phase 1 investigation (pass the confirmed context and the investigation brief). Wait for the Azure Findings Report.

If the user declines: note "Azure investigation skipped by user" in the final report. Do not continue the agent.

---

### Phase 3 — Synthesise and report

Wait for both agents to complete (or for the code agent alone if Azure was not applicable).

Present the consolidated Debug Report to the user:

```markdown
## Debug Report — <one-line issue title>

### Issue
<symptom from Phase 0 brief>

---

### Suspected Root Cause
<from the Bug Location Report>

### Evidence
<from the Bug Location Report>

### Data / Call Flow
<from the Bug Location Report>

### Related Tests
<from the Bug Location Report>

---

### Azure Findings
<verbatim Azure Findings Report from the azure-investigator agent — or "Not investigated (Azure not relevant)", "Skipped by user", "Not logged in — no active az CLI session">

---

### Open Uncertainties
<merged list from both agents>

---

> This report is read-only. No code or infrastructure was modified.
> Next step: decide how to address the root cause, then use `/project-implement-fix` or apply the fix manually.
```

---

## Conversation Style

- Be direct and factual — this is a diagnostic, not a design review.
- Do not propose solutions or hint at preferred fixes.
- If the agents contradict each other, surface both views rather than arbitrating.
- Use British English throughout.
