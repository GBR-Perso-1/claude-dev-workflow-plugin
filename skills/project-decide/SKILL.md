---
name: project-decide
description: Decision layer between investigation/debug and implementation. Reads findings from the conversation, generates 2–4 distinct solution options, evaluates each with pros/cons and effort signal, and commits to a recommended option grounded in long-term maintainability and architectural purity. Strictly read-only — no agents, no file I/O.
---

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Constraints

- **Read-only.** This skill must never modify source files, configuration, Azure resources, or any project artefact.
- **No agents.** This skill operates entirely from conversation context — it spawns no sub-agents and performs no file I/O.
- **No re-scanning.** The skill reasons over findings already present in the conversation. It does not re-run Glob, Grep, or Read operations against the codebase.
- **Option cap.** Generate a minimum of 2 and a maximum of 4 solution options.

## Input

`$ARGUMENTS` — optional short framing supplied by the user (e.g. "focus on the auth layer", "we can't touch the DB schema"). If absent, the skill infers scope from the investigation or debug report already in the conversation.

The skill's primary input is the **investigation or debug report already present in the conversation context**. It does not require the user to re-run a scan or re-paste findings.

---

### Phase 0 — Locate and absorb the report

1. Scan the current conversation context for an Investigation Report (produced by `/project-investigate`) or a Debug Report (produced by `/project-debug`). If both are present, prefer the most recent one.

2. If no report is found in the conversation, stop and inform the user:

   Use `AskUserQuestion`:
   - Question: "No investigation or debug report was found in the conversation. Would you like to run one first, or paste the findings directly?"
   - Options:
     - `Run /project-investigate first`
     - `Run /project-debug first`
     - `Paste findings now (I'll provide them)`

   If the user selects "Paste findings now", accept their free-text input as the findings and continue.

3. Read `$ARGUMENTS`. If provided, record it as the **user framing constraint** — it narrows the scope or excludes directions the user has ruled out. If absent, infer scope from the report content.

---

### Phase 1 — Restate the core problem

1. From the findings, derive a concise **Problem Statement** covering:
   - What the investigation or debug session determined is the core issue or opportunity
   - The affected area (layer, component, service) as evidenced by the report
   - Any constraints introduced by `$ARGUMENTS` (if provided)

2. Present the Problem Statement to the user and ask for confirmation via `AskUserQuestion`:
   - Question: "Before generating options, I want to confirm the problem we are solving. Does this capture it correctly?"
   - Present the Problem Statement in the question text.
   - Options:
     - `Yes — this is correct, generate options`
     - `No — let me correct it` (user provides correction via the free-text Other input)

3. If the user corrects it: incorporate their correction into the Problem Statement and proceed. Do not re-ask — one correction pass is sufficient.

---

### Phase 2 — Generate solution options

Generate between 2 and 4 distinct solution options by reasoning over the confirmed Problem Statement and the findings. Options must be meaningfully different directions — not variations of the same approach.

**Mandatory option types** (always include both, even if the option count is 2):
- At least one option must represent the **smallest safe change** — the minimum intervention needed to address the problem without restructuring.
- At least one option must represent the **most structurally correct** solution — the approach that best aligns with the architectural conventions and design principles observed in the investigation report.

If user framing constraints from `$ARGUMENTS` rule out a direction (e.g. "we can't touch the DB schema"), do not generate options that violate that constraint. Note the constraint in the relevant section.

For each option, produce all of the following:

```markdown
#### Option {N}: {Short title}

**Approach**
A plain-language description of what would change at the conceptual level. No code snippets. No file edit instructions. Describe the direction, not the mechanics.

**Pros**
- ...

**Cons**
- ...

**Effort**: Low / Medium / High *(relative to the other options in this report)*
```

Effort is always relative — calibrate across the full set of options generated, not against an absolute scale.

---

### Phase 3 — Evaluate and recommend

1. Review all options against the primary evaluation lens: **long-term maintainability and architectural purity**, where "architectural purity" is assessed relative to the conventions and patterns observed in the investigation report — not against an external standard.

2. Select one **Recommended Option**. State it clearly with a short rationale (2–4 sentences) grounded in the evaluation lens.

3. If a different option would be preferred under specific conditions (e.g. "if a quick hotfix is needed before the next release", "if the team decides to defer the schema change"), state that conditional preference explicitly alongside the main recommendation.

---

### Phase 4 — Produce the Decision Report

Present the full Decision Report to the user:

```markdown
## Decision Report — <one-line problem title>

### Problem
<confirmed Problem Statement from Phase 1>

---

### Options

#### Option 1: <title>

**Approach**
<plain-language description>

**Pros**
- ...

**Cons**
- ...

**Effort**: Low / Medium / High

---

#### Option 2: <title>
... (repeat for each option)

---

### Recommended Option

**Option {N}: <title>**

<rationale — 2–4 sentences grounded in long-term maintainability and architectural purity>

**Conditional preference**: <if a different option would be better under specific conditions, state it here — or omit this line if no conditional preference applies>

---

> This report is read-only. No code or infrastructure was modified.
> **Next step**: to implement the recommended option, use:
> - `/project-implement-fix` — for contained fixes and small changes
> - `/project-implement-new-features` — for larger structural changes or new capabilities
```

No confirmation gate is needed after presenting this report — it is a read-only analytical output. The user decides whether and how to act on it.

---

## Conversation Style

- Be analytical and opinionated — surface a clear recommendation, not a list of equally weighted choices.
- Ground every evaluation point in something specific from the findings (a pattern observed, a tradeoff identified, a convention in use).
- Do not propose implementation steps, code changes, or file edits — stay at the conceptual level.
- If the findings are ambiguous or incomplete, acknowledge the uncertainty in the relevant option's Cons rather than refusing to proceed.
- Use British English throughout.
