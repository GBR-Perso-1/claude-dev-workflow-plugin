---
name: project-investigate
description: Investigate an open-ended question about a codebase or system. Explores code and optionally Azure. Reports findings, patterns, tradeoffs, and areas to explore — never diagnoses bugs or proposes fixes.
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Constraints

- **Read-only.** This skill and every agent it spawns must never modify source files, configuration, or Azure resources.
- **No fixes.** Report findings only; the user decides how to act on them.
- **Not a bug-finder.** Look for patterns, bottlenecks, design tradeoffs, and usage characteristics — not defects.

## Input

`$ARGUMENTS` — a free-text description of the investigation question. If empty, ask the user to describe the question before proceeding.

---

### Phase 0 — Understand the question

1. Read `$ARGUMENTS`. If blank, stop and ask:
   Use `AskUserQuestion` to collect the investigation question:
   - Question: "What would you like to investigate? Describe your question — for example, a performance concern, a design tradeoff, how a feature works end-to-end, or any analytical question about the codebase or system."
   - Option: `I'll describe it` (user provides free-text via the automatic Other input)

2. Restate the question as a short investigation brief:
   - **Question** — what the user wants to understand
   - **Scope** — component, layer, or service (if mentioned)
   - **Azure involved?** — does the description mention infrastructure, auth, Key Vault, app registrations, Azure AD, Entra, MSAL, tenant, subscription, Static Web App, App Service, managed identity, deployments, or cloud resources?

3. Present the brief and ask the user to confirm or correct it using `AskUserQuestion`:
   - `Looks right — start investigating`
   - `Let me clarify` (user provides correction)

---

### Phase 0.5 — Quick scan

Using only Glob, Grep, and Read tools — without spawning any agents — perform a lightweight self-scan targeted at the investigation question from the confirmed brief.

1. Use `Glob` to locate files whose names or paths are likely relevant to the question's subject area (e.g. matching key nouns or component names from the brief).
2. Use `Grep` to search for key symbols, identifiers, or terms extracted from the brief across the codebase.
3. Read the most relevant 3–5 files or file sections identified by steps 1 and 2.
4. Produce a **Quick Findings** summary in this format:

```markdown
## Quick Findings — <one-line question title>

### What was found
<bullet list: relevant files, key symbols, patterns, or structural observations directly answering the question — grounded in specific file paths>

### Limitations of this scan
<what the quick scan could not determine — e.g. runtime behaviour, cross-service flows, Azure state>
```

Present the Quick Findings to the user, then ask via `AskUserQuestion`:

- **Question**: "Quick scan complete. Is this sufficient for your investigation?"
- **Options**:
  - `This is enough — I have what I need`
  - `Go deeper — run full investigation with agents`

If the user selects **"This is enough"**: the skill ends here. Do not proceed to Phase 1.

If the user selects **"Go deeper"**: proceed to Phase 1 below.

---

### Phase 1 — Parallel investigation

Launch **both agents simultaneously** when Azure is relevant. Otherwise launch only the code agent.

### 1a — Code investigation (always)

Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/repo-archaeologist.md`.

Prompt it with:

```
IMPORTANT: You are being used for targeted analytical investigation, not a repository overview. Override your standard output format entirely — do NOT write any file to .claude/reports/ and do NOT produce an archaeology report. Instead, produce only the Code Investigation Report format specified below.

You are investigating an analytical question about the codebase. Your goal is NOT to find bugs or propose fixes — only to identify patterns, bottlenecks, design tradeoffs, and usage characteristics relevant to the question.

Investigation brief:
<paste the brief from Phase 0>

Investigation tasks:
1. Identify the files, classes, and functions most relevant to the question.
2. Trace the relevant data flow, call chain, or execution path end-to-end.
3. Describe how the relevant code is structured: layers, responsibilities, coupling, cohesion.
4. Identify any performance characteristics visible from static analysis (query patterns, N+1 risks, caching, synchrony, resource use).
5. Note design tradeoffs: what is the current approach optimised for, and what does it trade away?
6. Flag any patterns that are notable, unusual, or that would surprise a new developer.

Output a structured Code Investigation Report (see format below). Do NOT suggest fixes.

---

## Code Investigation Report

### Summary
One paragraph. What the investigation question is about and what you found at a high level.

### Relevant Code Areas
Bullet list. For each area: file path + key locations, what it does, and why it is relevant to the question.

### Data / Call Flow
Step-by-step trace through the relevant path (file:line for each step where possible).

### Patterns & Characteristics
Bullet list. Each item describes an observable characteristic: a design pattern in use, a performance trait, a coupling, a cohesion property, or a notable convention.

### Design Tradeoffs
What the current approach is optimised for, and what it sacrifices. Grounded in specific code evidence.

### Uncertainties
What could not be determined from static analysis alone (e.g. runtime behaviour, load characteristics, external dependencies).
```

### 1b — Azure investigation (only when Azure is relevant)

Spawn the agent defined in `${CLAUDE_PLUGIN_ROOT}/agents/azure-investigator.md`.

Prompt it with:

```
You are investigating an Azure infrastructure question as part of an analytical investigation.
Your goal is NOT to find faults — only to characterise the Azure environment relevant to the question.

Investigation brief:
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

Present the consolidated Investigation Report to the user:

```markdown
## Investigation Report — <one-line question title>

### Question
<question from Phase 0 brief>

---

### Summary
<from the Code Investigation Report>

### Relevant Code Areas
<from the Code Investigation Report>

### Data / Call Flow
<from the Code Investigation Report>

### Patterns & Characteristics
<from the Code Investigation Report>

### Design Tradeoffs
<from the Code Investigation Report>

---

### Azure Findings
<verbatim Azure Findings Report from the azure-investigator agent — or "Not investigated (Azure not relevant)", "Skipped by user", "Not logged in — no active az CLI session">

---

### Open Uncertainties
<merged list from both agents>

---

> This report is read-only. No code or infrastructure was modified.
> Next step: decide how to act on these findings. If a change is needed, use `/project-implement-new-features` or `/project-implement-fix` as appropriate.
```

---

## Conversation Style

- Be analytical and neutral — surface what exists, not what should change.
- Do not propose solutions, fixes, or refactors.
- If the agents surface contradictory evidence, present both views.
- Use British English throughout.
