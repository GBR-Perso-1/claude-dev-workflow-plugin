# Dev Workflow — Claude Code Plugin

This repository is a general-purpose Claude Code plugin providing AI-assisted development workflow — agents, orchestration skills, project init, and language rules.

## Plugin Structure

- `skills/` — skills as `<name>/SKILL.md` + shared architecture docs in `skills/shared/`
- `agents/` — agent definitions (architect, developer, need analyst, tester, quality reviewers)
- `rules/` — rule files (copied to projects by `/project-init`, then auto-loaded natively by Claude Code)
- `hooks/` — `hooks.json` + `log-instructions-loaded.sh`
- `templates/` — `project-context.md` template (copied to projects by `/project-init`)
- `.claude-plugin/plugin.json` — plugin manifest

## How It Works

1. Install the plugin (user-level or project-level).
2. Skills and agents become available as slash commands (`/project-session`, `/project-init`, etc.).
3. Rules are copied to projects by `/project-init` (into `.claude/rules/`), where Claude Code auto-loads them natively based on `paths:` frontmatter. Run `/project-init` again to refresh rules after updating them here.
4. The `InstructionsLoaded` hook logs which instruction files are loaded to `.claude/instructions-loaded.log`.

## Development

When editing this repo, you are editing the plugin source. Changes here affect all consumer projects.

- Skills reference shared docs via `${CLAUDE_PLUGIN_ROOT}/skills/shared/` — keep `shared/` as a sibling of skill directories.
- Rules use YAML frontmatter `paths:` for file-type scoping — Claude Code auto-loads them natively; no skill involvement needed.
- Agent references in skills must use `${CLAUDE_PLUGIN_ROOT}/agents/<name>.md` — never `.claude/agents/` (that's the consumer project, not the plugin).
- Test skill changes by running them in a consumer project after updating the plugin.
