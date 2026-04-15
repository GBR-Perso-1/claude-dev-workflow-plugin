# Dev Workflow Plugin — Plugin-Aware Implement Skill

## Vision

The plugin is now actively developed at pace — several artefacts per month. The existing
`project-implement` skill works technically, but it is designed for app-stack development
(C# API, Vue SPA, Terraform) and carries assumptions that are misaligned with plugin
artefacts (SKILL.md files, agent definitions, hooks, rules). More critically, it has no
awareness of the plugin ecosystem: it cannot flag conflicts with existing skills, missing
agent dependencies, or opportunities to extend rather than duplicate.

The goal is a dedicated plugin implementation skill that treats the plugin itself as the
product — loading the existing ecosystem before doing any work, handling all plugin artefact
types, and reviewing output against plugin conventions rather than app-stack conventions.

## Epics

### Epic 1: Plugin-Implement Skill

**Goal**: A single skill that implements new or updated plugin artefacts (skills, agents,
hooks, rules) with full awareness of the existing plugin ecosystem — fast, integrated,
and convention-compliant.

#### Requirements

- **REQ-1.1**: The skill accepts a natural-language description of what needs to be built
  or changed (via skill arguments or an initial prompt).

- **REQ-1.2**: Before any design work, the skill surveys the entire existing plugin —
  all skills, agents, hooks, and rules — and produces a concise summary of what currently
  exists, so the design phase is fully informed.

- **REQ-1.3**: During the design phase, the skill actively flags:
  - Conflicts or duplicates (the proposed artefact overlaps significantly with an existing one)
  - Missing dependencies (the plan references an agent, shared doc, or hook trigger that
    does not yet exist)
  - Reuse opportunities (an existing artefact could be extended rather than a new one created)

- **REQ-1.4**: The skill determines the full set of artefacts required — a request may
  result in just a skill, or a combination of skill + agent(s) + rule(s) + hook entries.
  It does not assume a fixed output shape.

- **REQ-1.5**: The skill implements all planned artefacts using plugin conventions:
  - SKILL.md files with correct structure, argument handling, and `${CLAUDE_PLUGIN_ROOT}/`
    references
  - Agent `.md` files with correct frontmatter and tool declarations
  - `hooks.json` additions with correct event names and script references
  - Rule `.md` files with correct `paths:` frontmatter for file-type scoping

- **REQ-1.6**: The skill handles both creating new artefacts and updating existing ones.
  It detects from context whether it is bootstrapping something new or revising something
  that already exists, and adjusts accordingly.

- **REQ-1.7**: After implementation, the skill validates the output against plugin
  conventions — not app-stack conventions. Validation covers:
  - Format compliance (SKILL.md structure, agent frontmatter, hooks.json schema)
  - Internal consistency (skill references agents that exist, agents reference tools that
    are valid)
  - Cross-artefact consistency (no duplicate skill names, no conflicting trigger names)

- **REQ-1.8**: The plugin manifest (`plugin.json`) is updated if the new artefacts
  need to be registered there.

#### Decisions & Assumptions

- The skill name follows the existing pattern (`plugin-implement` or similar) — exact
  name TBD at implementation time.
- `project-implement` is not modified — it remains the right tool for app-stack development.
  The two skills coexist with different target domains.
- No brainstorm phase is included — for plugin artefacts, the design intent is clear
  enough before running the skill. The ecosystem-awareness step replaces the need for
  a separate brainstorm.
- The skill lives in this plugin repo and is self-hosted — it is used to develop the
  plugin from within the plugin.

## Priorities

| Priority | Epic / Requirement | Rationale |
| -------- | ------------------ | --------- |
| 1 | REQ-1.2 — Ecosystem survey | This is the core differentiator; without it the skill is just a reformatted project-implement |
| 2 | REQ-1.3 — Conflict/gap/reuse detection | Directly addresses the "warn gap error conflict" need |
| 3 | REQ-1.4 — Variable output shape | A skill may need agents, hooks, rules — not just a SKILL.md |
| 4 | REQ-1.5 — Plugin-convention implementation | Correct format output |
| 5 | REQ-1.6 — Create and update | Update support needed but can be partial in v1 |
| 6 | REQ-1.7 — Plugin-aware review | Quality gate — valuable but can be lightweight initially |
| 7 | REQ-1.8 — Manifest update | Lower risk; plugin.json may not always need updating |

## Out of Scope

- A `/plugin-brainstorm` skill — not needed; design intent is clear before implementation.
- Modifying `project-implement` or `project-brainstorm` — they remain app-stack focused.
- Automated testing of generated skill/agent prompts — prompt quality is reviewed by
  the plugin-aware reviewer, not run through a test suite.
- Publishing or distributing plugin artefacts to other users — out of scope for this skill.
