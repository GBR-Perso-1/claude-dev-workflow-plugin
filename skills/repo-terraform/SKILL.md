---
name: repo-terraform
description: "Smart Terraform apply pipeline: validate → plan → analyze risks → confirm → apply. Environment-aware safety gates."
---

Smart Terraform deployment pipeline that validates, plans, analyzes risks, and applies — with environment-appropriate safety gates.

## When invoked without arguments

If `$ARGUMENTS` is empty, print usage and stop:

```
**Terraform Deploy**

Validates, plans, analyzes, and applies Terraform changes with safety gates.

Usage:
  /repo-terraform apply <environment>

  <environment>   "qa" or "prod"

Examples:
  /repo-terraform apply qa
  /repo-terraform apply prod
```

Then stop and wait.

## Arguments

`$ARGUMENTS`: `apply <environment>`

- The first word must be `apply`. If it's anything else, print usage and stop.
- The second word is the environment: `qa` or `prod`. If missing or invalid, ask for it.

## Step 0: Auth file check

**This is the very first check — run it before anything else.**

The project's `infra/scripts/terraform-run.ps1` uses one of two auth formats. Detect which by grepping the script:

- If it references `rise-env-` → **user auth (new)**: looks for `%USERPROFILE%\.azure\rise-env-<env>.env` (tenant + subscription, no secrets; MFA login performed by the script).
- If it references `-credentials-` → **service-principal auth (legacy)**: looks for `%USERPROFILE%\.azure\*-credentials-<env>.ps1` (full SP creds).

### User auth (new)

Check `%USERPROFILE%\.azure\rise-env-<env>.env` exists.

**Found** — announce and continue:
```
Using user auth: rise-env-<env>.env
```

**Missing** — stop:
```
No env file found for environment: <env>

Create %USERPROFILE%\.azure\rise-env-<env>.env with:

  ARM_SUBSCRIPTION_ID=<Azure_Subscription_ID>
  ARM_TENANT_ID=<Azure_Tenant_ID>

The script performs interactive `az login` (MFA) against this tenant — no client secret is stored on disk.
```

### Service-principal auth (legacy)

Discover credential files:

```bash
ls "$USERPROFILE/.azure/"*"-credentials-<env>.ps1" 2>/dev/null
```

**No files found** — stop:
```
No credential file found for environment: <env>

Create %USERPROFILE%\.azure\<tenant>-credentials-<env>.ps1 with:

  $env:ARM_CLIENT_ID = "<Azure_Client_ID>"
  $env:ARM_CLIENT_SECRET = "<Azure_Client_Secret>"
  $env:ARM_TENANT_ID = "<Azure_Tenant_ID>"
  $env:ARM_SUBSCRIPTION_ID = "<Azure_Subscription_ID>"

Where <tenant> is a short name for the Azure tenant (e.g. "ekla", "perso").
```

**Exactly one file found** — use it and announce:
```
Using tenant: <tenant> (<full-filename>)
```

**Multiple files found** — list them and ask the user which tenant to use via `AskUserQuestion`. Wait before proceeding.

Do NOT proceed to the structure check or pipeline until the auth file is resolved.

## Structure sanity check

Before doing anything else, verify the project matches this layout:

```
infra/
  scripts/
    terraform-run.ps1
    terraform-destroy.ps1
  terraform/
    main.tf
    variables.tf
    providers.tf
    backend.tf
    _<env>.tfvars
    _<env>-backend.tfvars
    modules/
      ...
```

Check **all** of the following. If any check fails, print the full expected structure above, mark which items are missing with `[MISSING]`, and stop with a clear message. Do NOT proceed to the pipeline.

1. **`infra/terraform/` directory** — must exist at the repo root.
2. **`infra/scripts/terraform-run.ps1`** — the unified runner script.
3. **`infra/terraform/main.tf`** — there must be at least a root Terraform config.
4. **`infra/terraform/backend.tf`** — backend must be configured.
5. **`infra/terraform/_<env>.tfvars`** — environment variables file for the target environment.
6. **`infra/terraform/_<env>-backend.tfvars`** — backend config for the target environment.

Example failure output:

```
This project does not match the expected Terraform structure.

infra/
  scripts/
    terraform-run.ps1              [OK]
  terraform/
    main.tf                        [OK]
    variables.tf                   [OK]
    providers.tf                   [OK]
    backend.tf                     [OK]
    _qa.tfvars                     [OK]
    _qa-backend.tfvars             [MISSING]

Fix the missing items and try again.
```

Only proceed to the pipeline if all checks pass.

## Pipeline

### Step 1: Validate

Run:
```bash
cd infra/terraform && pwsh -NoProfile ../scripts/terraform-run.ps1 -Environment <env> -Mode validate 2>&1
```

- Capture all output.
- **If it fails** (non-zero exit): show the error output and stop. Tell the user what's wrong so they can fix it. Do NOT proceed.
- **If it succeeds**: print a short confirmation and move to step 2.

### Step 2: Plan

Run:
```bash
cd infra/terraform && pwsh -NoProfile ../scripts/terraform-run.ps1 -Environment <env> -Mode plan -PlanFile plan.tfplan 2>&1
```

- Capture all output.
- **If it fails**: show the error output and stop.
- **If it succeeds**: move to step 3 with the captured output.

### Step 3: Analyze the plan

Parse the plan output to extract:

1. **Summary counts** — look for the line:
   ```
   Plan: X to add, Y to change, Z to destroy.
   ```
   or:
   ```
   No changes. Your infrastructure matches the configuration.
   ```

2. **Resources being destroyed** — scan for lines containing `will be destroyed` or `must be replaced`. Extract the full resource address for each.

3. **Resources being added** — scan for lines containing `will be created`. Extract up to 10 resource addresses.

4. **Resources being changed** — scan for lines containing `will be updated in-place`. Extract up to 10 resource addresses.

#### Risk assessment

Classify the plan:

- **NO CHANGES**: nothing to do. Print "No changes detected — infrastructure matches configuration." and **stop** (no apply needed).
- **LOW RISK**: only adds and/or in-place updates, zero destroys.
- **MEDIUM RISK**: has destroys, but they are expected removals (e.g. a resource was removed from config on purpose). Any destroy counts as medium risk minimum.
- **HIGH RISK**: has replacements (`must be replaced`) — Terraform will destroy and recreate a resource. This can cause downtime or data loss.

#### Present the summary

```
──────────────────────────────────────
  TERRAFORM PLAN — <ENV>
──────────────────────────────────────

  | Action  | Count |
  |---------|-------|
  | Add     | X     |
  | Change  | Y     |
  | Destroy | Z     |

  Risk: LOW / MEDIUM / HIGH

  [If MEDIUM or HIGH, list every resource being destroyed or replaced:]
  Destroyed/replaced resources:
    - azurerm_storage_account.main (DESTROY)
    - azurerm_linux_web_app.api (REPLACE — will be destroyed and recreated)

  [If LOW, list up to 10 notable adds/changes:]
  Notable changes:
    - azurerm_windows_web_app.api (update in-place)
    - azurerm_static_web_app.frontend (create)

──────────────────────────────────────
```

### Step 4: Gate — ask for confirmation

The confirmation prompt depends on the environment and risk:

**QA — any risk level:**
```
Confirm apply to QA? (yes/no)
```

**Prod — LOW risk:**
```
Type APPLY PROD to confirm, or anything else to cancel.
```

**Prod — MEDIUM or HIGH risk:**
```
This plan includes DESTRUCTIVE changes (see above).

Type APPLY PROD to confirm, or anything else to cancel.
```

Wait for the user's response:
- **QA**: accept `yes`, `y`, or `YES`. Anything else = cancel.
- **Prod**: accept exactly `APPLY PROD` (case-sensitive). Anything else = cancel.

If cancelled, print "Apply cancelled." and **stop**.

### Step 5: Apply

Run:
```bash
cd infra/terraform && pwsh -NoProfile ../scripts/terraform-run.ps1 -Environment <env> -Mode apply -PlanFile plan.tfplan 2>&1
```

- Capture all output.
- **If it fails**: show the error output. Tell the user what failed.
- **If it succeeds**: print a final summary:

```
──────────────────────────────────────
  APPLY COMPLETE — <ENV>
──────────────────────────────────────
  Added:     X
  Changed:   Y
  Destroyed: Z
──────────────────────────────────────
```

## Guardrails

- **Never pass `-AutoApprove`** to any Terraform command — the plan file is the approval mechanism.
- **Never run `terraform destroy`** — this skill does not support full infrastructure destruction. Users must run `terraform-destroy.ps1` manually for that.
- **Always plan before apply** — never apply without generating and analyzing a plan first.
- **Always validate before plan** — catch config errors early.
- **Never skip the gate** — even if the user says "just do it", always show the plan analysis and require confirmation.
- **Never show secret values** — if the plan output contains sensitive values, they should already be masked by Terraform. Do not attempt to print raw tfvars content.
- **Run commands from the correct directory** — always `cd infra/terraform` before running the script.
