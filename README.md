# Dev Workflow — Claude Code Plugin

General-purpose Claude Code plugin providing AI-assisted development workflow — agents, orchestration skills, project init, and language rules.

## What's included

| Component  | Items                                                                                                                                                     |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Skills** | `project-init`, `project-session`, `project-brainstorm`, `project-implement-new-features`, `project-implement-fix`, `repo-commit`, `repo-terraform`, `repo-git-env`, `repo-ef-sql`, `repo-tidy` |
| **Agents** | `architect`, `developer`, `test-writer`, `reviewer-quality`, `reviewer-design`, `reviewer-perf`, `codebase-explorer`                                      |
| **Hooks**  | `InstructionsLoaded` — logs loaded instruction files to `.claude/instructions-loaded.log`                                                                 |
| **Rules**  | `general`, `api-lang`, `app-lang`, `infra-lang`, `infra-naming`, `python-lang` — copied to consumer projects by `/project-init`                           |

---

## Installation

### Prerequisites

- [Claude Code](https://code.claude.com) installed

### Step 1 — Register the marketplace

Add the following to your **user-level** settings file at `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "ekla-internal": {
      "source": {
        "source": "settings",
        "name": "ekla-internal",
        "owner": {
          "name": "Ekla"
        },
        "plugins": [
          {
            "name": "dev-workflow",
            "source": {
              "source": "url",
              "url": "https://github.com/azuregbr/claude-dev-workflow-plugin.git"
            },
            "description": "AI-assisted development workflow — agents, skills, project init, and language rules.",
            "version": "1.0.0"
          }
        ]
      }
    }
  }
}
```

### Step 2 — Install the plugin

```powershell
claude plugin install dev-workflow@ekla-internal --scope user
```

### Step 3 — Reload plugins

```
/reload-plugins
```

### Step 4 — Bootstrap a consumer project

In any project, run:

```
/project-init
```

This copies the relevant rule files into `.claude/rules/` and creates `.claude/CLAUDE.md` and `.claude/project-context.md`.

---

## Usage

| Skill                 | Purpose                                                                |
| --------------------- | ---------------------------------------------------------------------- |
| `/project-init`       | Bootstrap or refresh a project's `.claude/` directory                  |
| `/project-session`    | Load full architectural context for a deep-work session                |
| `/project-brainstorm` | Brainstorm and write a requirements document                           |
| `/project-implement-new-features`  | Implement a feature using the architect → dev → test → review pipeline |
| `/project-implement-fix`           | Quick fix — skill interprets request, dev implements, short test loop, no reviewers, no commit |
| `/repo-commit`        | Run tests, commit, and push                                            |
| `/repo-terraform`     | Validate → plan → analyze risks → confirm → apply Terraform            |
| `/repo-git-env`       | Manage GitHub environment variables and secrets                        |
| `/repo-ef-sql`        | Generate a SQL upgrade script from an EF Core migration                |
| `/repo-tidy`          | Clean up `.claude/settings.json`                                       |

---

## Updating the plugin

```powershell
claude plugin update dev-workflow@ekla-internal
```

---

## Local development

```powershell
claude --plugin-dir "C:/Workspace/Dev/Perso.Applications/claude-dev-workflow-plugin"
```
