---
name: repo-git-env
description: Manage GitHub environment variables and secrets for a repository from a JSON config file — validates, confirms, and applies directly via gh CLI. Can also scan a project codebase and generate a template config with placeholder values.
---

Manage GitHub environment variables and secrets for a repository from a JSON config file.

The skill validates the config, verifies the target repo, shows a dry-run summary, asks for explicit confirmation, then applies changes directly via `gh` CLI. No intermediate script is generated — secrets never touch disk.

## When invoked without arguments

If `$ARGUMENTS` is empty, print a short usage guide and wait for user input — do **not** proceed to the steps below. Example output:

```
**GitHub Environment Setup**

Sets (or removes) GitHub environment variables and secrets from a JSON config.

Usage:
  /repo-git-env <repo> <config.json>       — apply config to repo
  /repo-git-env generate [project-path]     — scan codebase, generate template

  <repo>          owner/repo or full GitHub URL
  <config.json>   path to the env config file (see template below)

Examples:
  /repo-git-env my-org/my-repo ./env-setup.json
  /repo-git-env generate .
  /repo-git-env generate C:/Workspace/Dev/my-project

Config template: ${CLAUDE_PLUGIN_ROOT}/skills/repo-git-env/env-config.template.json

Special values:
  @file:./path   — read secret value from a file at apply time
  @remove        — delete the variable or secret from the environment
```

Then stop and wait for the user to provide arguments.

## Arguments

`$ARGUMENTS`: either:
- `generate [project-path]` — scan a project codebase and generate an `env-config.json` template (see **Generate mode** below)
- `<repo-url-or-owner/repo> <path-to-config.json>` — apply config to a repo (see **Apply mode** below)

If the first argument is `generate`, follow the **Generate mode** section.
Otherwise, follow the **Apply mode** section.

If only one argument is provided and it's not `generate`, ask for the missing one.

## Guardrails

- **Never log secret values** in output — only print key names and masked placeholders (`****`)
- **Never write secrets to disk** — no generated scripts, no temp files
- The JSON config file must exist and be valid JSON
- Reject configs with empty secret values (print which keys are empty and stop)
- For `@file:` references, verify the referenced file exists before proceeding
- Removals must be **explicit** — only delete a variable or secret when its value is `"@remove"` in the config
- **Always verify the target repo via `gh repo view`** before showing the summary — abort if the repo doesn't exist or the user lacks access
- **Never proceed without explicit user confirmation** — the user must type YES after reviewing the summary

## JSON Config Format

```json
{
  "environments": {
    "qa": {
      "variables": {
        "AZURE_WEBAPP_NAME": "app-qa",
        "APP_URL": "https://qa.example.com",
        "OLD_SETTING": "@remove"
      },
      "secrets": {
        "APPSETTINGS_JSON": "@file:./secrets/qa/appsettings.json",
        "AZURE_DEPLOYMENT_TOKEN": "token-qa",
        "LEGACY_KEY": "@remove"
      }
    },
    "prod": {
      "variables": {
        "AZURE_WEBAPP_NAME": "app-prod",
        "APP_URL": "https://example.com"
      },
      "secrets": {
        "APPSETTINGS_JSON": "@file:./secrets/prod/appsettings.json",
        "AZURE_DEPLOYMENT_TOKEN": "token-prod"
      }
    }
  }
}
```

- `environments`: keyed by environment name (`qa`, `prod`, etc.), each with `variables` and `secrets`

### File references (`@file:`)

When a value starts with `@file:`, it means "read the secret value from this file path". This is used for secrets whose value is an entire file (e.g. `appsettings.json`, XML publish profiles).

- The path is relative to the config file's directory
- The referenced file must exist (validation step)
- In the summary, show as `@file:./path` (never print file contents)
- At apply time, read the file content and pipe it directly to `gh secret set` — never store it in a variable or print it

This avoids having to escape JSON-inside-JSON or large XML blobs.

### Removal marker (`@remove`)

When a value is exactly `"@remove"`, the variable or secret will be **deleted** from the environment.

- Works for both `variables` and `secrets`
- In the summary, show removals on dedicated `REMOVE:` lines so they stand out

## Generate mode

When the first argument is `generate`, scan a project's codebase to discover which GitHub environment variables and secrets it expects, then write an `env-config.json` with placeholder values.

### Generate 1. Resolve project path

- If a path is given after `generate`, use it. Otherwise use the current working directory.
- Verify the path exists and looks like a project root (has `.github/` or `src/` or a `.sln`/`package.json`).

### Generate 2. Scan the codebase for variable and secret references

Scan these sources in order, collecting variable and secret names per environment:

1. **GitHub workflow files** (`.github/workflows/*.yml`):
   - Extract `vars.<NAME>` references → add to `variables`
   - Extract `secrets.<NAME>` references → add to `secrets`
   - Detect environment names from `environment:` fields in jobs (e.g. `qa`, `prod`)
   - Ignore `secrets.GITHUB_TOKEN` (built-in, not user-managed)

2. **Existing `.github/appsettings-value/` directory** (if present):
   - List subdirectories as environment names
   - For each environment subdir containing an `appsettings.json`, add `APPSETTINGS_JSON` to that environment's `secrets` with value `@file:./secrets/<env>/appsettings.json`

3. **Terraform files** (`terraform/*.tf`, `infra/*.tf`):
   - Look for `var.` references that map to GitHub variables (e.g. variables passed via `-var` in workflow files)

### Generate 3. Assign placeholder values

For each discovered entry, assign a sensible placeholder:

- **Variables**: use `"<VARIABLE_NAME>"` as placeholder (e.g. `"<AZURE_WEBAPP_NAME>"`)
- **Secrets with `@file:` pattern** (like `APPSETTINGS_JSON`): use `"@file:./secrets/<env>/appsettings.json"`
- **Other secrets**: use `"<SECRET_NAME>"` as placeholder (e.g. `"<AZURE_DEPLOYMENT_TOKEN>"`)

If no environments were detected, default to `qa` and `prod`.

If a variable or secret appears in workflows without a specific environment context (e.g. referenced in a job with no `environment:` field), include it in **all** detected environments.

### Generate 4. Write the template file

Write the generated JSON to `<project-path>/.github/env-config.json`.

Before writing, show the user a preview of the generated JSON and the output path. Ask for confirmation before writing.

If the file already exists, warn the user and ask whether to overwrite.

### Generate 5. Print next steps

After writing, print:

```
Template written to <path>/.github/env-config.json

Next steps:
  1. Fill in the placeholder values (search for < and >)
  2. For @file: secrets, place the actual files at the referenced paths
  3. Apply with: /repo-git-env <owner/repo> <path>/.github/env-config.json
```

---

## Apply mode

The following steps apply when the skill is invoked with `<repo> <config.json>` arguments.

### 1. Parse arguments

Extract the repo identifier and config file path from `$ARGUMENTS`.

Accept repo as either:

- Full URL: `https://github.com/my-org/my-repo` → extract `my-org/my-repo`
- Short form: `my-org/my-repo`

### 2. Verify the target repo

Run `gh repo view <owner/repo> --json nameWithOwner,url` to confirm:

- The repo exists
- The user has access
- The resolved `nameWithOwner` matches what was provided

If the repo doesn't exist or access is denied, print the error and **stop**.

### 3. Read and validate the config

Read the JSON config file. Validate:

- File exists
- Valid JSON
- Has `environments` with at least one environment
- No empty string values in `secrets` or `variables` (print which keys are empty and stop)
- All `@file:` references point to files that exist (print which are missing and stop)
- `@remove` values are valid in both `variables` and `secrets` — they mark items for deletion

### 4. Show dry-run summary and ask for confirmation

Print the verified repo prominently, then the summary table (secrets masked, file refs shown as paths):

```
──────────────────────────────────────
  TARGET REPO: my-org/my-repo
  https://github.com/my-org/my-repo
──────────────────────────────────────

  [qa]
    SET variables:  AZURE_WEBAPP_NAME = app-qa, APP_URL = https://qa.example.com
    SET secrets:    APPSETTINGS_JSON = @file:./secrets/qa/appsettings.json
                    AZURE_DEPLOYMENT_TOKEN = ****
    REMOVE:         OLD_SETTING (variable), LEGACY_KEY (secret)

  [prod]
    SET variables:  AZURE_WEBAPP_NAME = app-prod, APP_URL = https://example.com
    SET secrets:    APPSETTINGS_JSON = @file:./secrets/prod/appsettings.json
                    AZURE_DEPLOYMENT_TOKEN = ****

  Total: 4 variables to set, 4 secrets to set, 1 variable to remove, 1 secret to remove
──────────────────────────────────────
```

Then ask:

```
Type YES to apply these changes to my-org/my-repo, or anything else to cancel.
```

**Do NOT proceed unless the user responds with exactly `YES`.** Any other response = cancel.

### 5. Apply changes

Execute `gh` commands one by one, printing progress for each:

1. **Create environments** if they don't exist:
   ```
   gh api --method PUT "repos/<owner/repo>/environments/<env-name>"
   ```

2. **Set variables** (non-`@remove`):
   ```
   gh variable set <NAME> --body "<value>" --repo <owner/repo> --env <env>
   ```

3. **Set secrets** with plain values (non-`@file:`, non-`@remove`):
   ```
   gh secret set <NAME> --body "<value>" --repo <owner/repo> --env <env>
   ```

4. **Set secrets** with `@file:` references — read and pipe directly:
   ```bash
   cat "<resolved-path>" | gh secret set <NAME> --repo <owner/repo> --env <env>
   ```

5. **Remove variables** marked `@remove`:
   ```
   gh variable delete <NAME> --repo <owner/repo> --env <env> --yes
   ```

6. **Remove secrets** marked `@remove`:
   ```
   gh secret delete <NAME> --repo <owner/repo> --env <env> --yes
   ```

For each command, print a status line:

```
  [qa] SET variable AZURE_WEBAPP_NAME ✓
  [qa] SET secret APPSETTINGS_JSON ✓
  [qa] REMOVE variable OLD_SETTING ✓
  ...
```

If any command fails, print the error for that item and **continue with the remaining items** (don't abort the whole batch). At the end, print a final summary:

```
Done. 8 succeeded, 0 failed.
```

If there were failures, list them so the user can retry manually.
