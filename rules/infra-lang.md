---
paths:
  - "**/*.tf"
  - "**/*.tfvars"
  - "**/terraform*"
---

# Terraform / HCL Language Rules

## Naming
- Resources: snake_case, descriptive (`azurerm_resource_group.estate_ops_rg`)
- Variables: snake_case with clear purpose
- Outputs: snake_case matching the resource they expose
