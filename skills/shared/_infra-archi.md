---
paths:
  - "infra/**"
---

# Infrastructure Architecture Rules

## Structure
- modules/ — reusable modules
- environments/ — env-specific configs (dev, staging, prod)
- One resource type per file (networking.tf, compute.tf, storage.tf)

## Practices
- Remote state backend (Azure Storage / S3)
- Lock state files
- Use variables for anything environment-specific
- Tag all resources with: project, environment, managed_by=terraform
- Never hardcode secrets — use Key Vault / Secrets Manager references

## Safety
- **NEVER** run `terraform apply` against production without explicit confirmation in the current session
- Always run `terraform plan` first and wait for user review, regardless of environment
- Never run `terraform destroy` on QA or production without explicit confirmation
- Never commit credentials files — credentials live in `%USERPROFILE%\.azure\<tenant>-credentials-<env>.ps1` (e.g. `ekla-credentials-prod.ps1`, `perso-credentials-dev.ps1`), never inside a repo. If multiple credential files exist for different tenants, ask the user which tenant to target before proceeding.
- Never invent a resource type shorthand silently — propose one and wait for confirmation
- Ask before modifying `main.tf` or any module that affects production resources
