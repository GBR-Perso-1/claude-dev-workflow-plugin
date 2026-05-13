---
name: project-session
description: "Load project reference material and current session context for a productive development session. Run at the start of a conversation to give Claude full project awareness."
---

Load project reference material and current session context for a productive development session.

## Important rules

Read and follow all rules in [`../shared/_ux-rules.md`](../shared/_ux-rules.md).

## Architecture Docs — Load at Session Start

Detect which stacks are present and read the relevant architecture docs into context:

| Detection                 | Architecture doc                                       |
| ------------------------- | ------------------------------------------------------ |
| `api/` directory exists   | `${CLAUDE_PLUGIN_ROOT}/skills/shared/_api-archi.md`    |
| `app/` directory exists   | `${CLAUDE_PLUGIN_ROOT}/skills/shared/_app-archi.md`    |
| `infra/` directory exists | `${CLAUDE_PLUGIN_ROOT}/skills/shared/_infra-archi.md`  |
| Any `**/*.py` files exist | `${CLAUDE_PLUGIN_ROOT}/skills/shared/_python-archi.md` |

## Instructions

1. Run `git branch --show-current` and `git status --short` to capture the current branch and working tree state.
2. Present the session context, then the reference material below — formatted clearly, no commentary needed.

---

## Session Context

- **Branch**: _(from step 1)_
- **Uncommitted changes**: _(from step 1, or "clean")_

---

## Backend API (`api/`)

> Only present this section if `api/` exists in the project.

### Commands

```bash
dotnet build api/{{solutionName}}.sln
dotnet test api/{{solutionName}}.sln
dotnet run --project api/src/Api                              # https://localhost:7006
dotnet test api/{{solutionName}}.sln --filter "FullyQualifiedName~ClassName"
dotnet test api/{{solutionName}}.sln --filter "FullyQualifiedName~ClassName.Method"
```

EF Migrations (from repo root):

```bash
dotnet ef migrations add <name> --project api/src/Infrastructure --startup-project api/src/Api --output-dir Data/Migrations --context ApplicationDbContext
dotnet ef database update --project api/src/Infrastructure --startup-project api/src/Api --context ApplicationDbContext
dotnet ef migrations script <from> <to> --project api/src/Infrastructure --startup-project api/src/Api --context ApplicationDbContext -o sql-scripts/_latest-upgrade-db.sql
```

User secrets (from `api/src/Api/`): `dotnet user-secrets set "AZURE_STORAGE_KEY" "<key>"`

### Project Details

- **Domain entities**: `AccessRight`, `NavigationItem`, `BackgroundTask`
- **Base classes**: `BaseEntity` (int Id + domain events), `BaseAuditableEntity` (+audit), `BaseEvent`
- **Pipeline behaviours**: `UnhandledExceptionBehaviour` → `ValidationBehaviour`
- **`IApplicationService`**: auto-registered in DI
- **`ApplicationDbContext`**: schema `{{dbSchema}}`, collation `SQL_Latin1_General_CP1_CI_AS`, migrations auto-run on startup
- **Interceptors**: `AuditableEntityInterceptor`, `DispatchDomainEventsInterceptor`
- **Cache**: `AppMemoryCache` (singleton)
- **Auth**: Entra ID JWT Bearer; `DummyAuthenticationHandler` when `OfflineAuthBypass=true`. Attributes: `[SysAdminAuthorise]`, `[AdminAuthorise]`, `[UserAuthorise]`
- **API docs**: OpenAPI + Scalar UI at `/scalar/v1`; Health: `/health`; Output cache: 1h default, 30s policies

### Testing

- xunit + FluentAssertions + Moq + Respawn + Bogus
- Integration tests use LocalDB (auto-created on first run)
- `SharedFixture`: collection fixture for DI, DB, mocks
- `IntegrationTest` base: resets DB (Respawn) and cache before each class
- Test classes extend `IntegrationTest(sharedFixture)`, use `SharedFixture.SendAsync()`

---

## Frontend App (`app/`)

> Only present this section if `app/` exists in the project.

### Commands

```bash
npm run dev             # localhost:3500
npm run build
npm run lint
npm run prettier
npm run type-check
npm run test
npm run nswag:generate
```

### Project Details

- **Quasar defaults** in `app/src/boot/DEFAULT_SETTINGS.json`
- **Routing**: Navigation from backend BFF API, built dynamically via `RouteBuilder`
- **Dark mode**: Quasar `Dark` plugin + localStorage

---

## Integration Points

> Only present this section if both `api/` and `app/` exist in the project.

### API client generation

Frontend `ApiClient.ts` is auto-generated from the Backend OpenAPI spec (NSwag).
Reads from `api/src/Api/_openapi.spec/openapi.json` — no running API needed.

1. Build the API: `dotnet build api/{{solutionName}}.sln`
2. Generate: `nswag run .\.project\{{nswagConfig}}.nswag /runtime:Net100` (or from `app/`: `npm run nswag:generate`)

Config: `.project/{{nswagConfig}}.nswag` — Axios template, output to `app/src/services/ApiClient.ts`.
Regenerate whenever the Backend API contract changes.

### Authentication chain

- **Frontend → API**: MSAL acquires Entra ID JWT → Bearer token via Axios
- **API** validates tokens (or bypasses with `OfflineAuthBypass=true` in dev)

### Environments

| Env         | Backend API       | Frontend             | Infra tfvars   |
| ----------- | ----------------- | -------------------- | -------------- |
| Development | `localhost:7006`  | `localhost:3500`     | —              |
| QA          | Azure App Service | Azure Static Web App | `_qa.tfvars`   |
| Production  | Azure App Service | Azure Static Web App | `_prod.tfvars` |

CI/CD: GitHub Actions. Backend → App Service, Frontend → Static Web Apps. Infra deployment is manual via PowerShell.

---

## Infrastructure (`infra/`)

> Only present this section if `infra/` exists in the project.

### Resource Type Shorthands

| Short   | Resource Type   | Short | Resource Type        |
| ------- | --------------- | ----- | -------------------- |
| `rg`    | Resource Group  | `sa`  | Storage Account      |
| `api`   | API App Service | `ai`  | Application Insights |
| `app`   | Frontend (SWA)  | `ar`  | App Registration     |
| `db`    | Database        | `kv`  | Key Vault            |
| `dbsrv` | Database Server | `fa`  | Function App         |
| `sp`    | Service Plan    |       |                      |

If a shorthand doesn't exist here, **STOP** and ask.

### Project Details

- Project name: 12 chars max (storage account limit)
- Naming: `<prefix><project><type><env><index>` (e.g. `{{infraPrefix}}rgprod01`)

### Modules

- **web_app**: App Service + Static Web App + AD app registrations (roles: user/admin/sys-admin, scope: `{project}.access.full`)
- **storage**: Storage Account + document container + RBAC (Blob Data Contributor → managed identity)
- **Root** (`main.tf`): resource group → web_app → storage

### Commands

All from `infra/scripts/`, accept environment param (`qa`, `prod`):

```powershell
.\terraform-validate.ps1 -Environment "qa"
.\terraform-plan.ps1 -Environment "qa"
.\terraform-apply.ps1 -Environment "qa"
.\terraform-destroy.ps1 -Environment "qa"
```

### First-time setup

1. Create `%USERPROFILE%\.azure\<tenant>-credentials-prod.ps1` (and `-qa.ps1`)
2. Fill in Azure SP credentials (ARM_CLIENT_ID, ARM_CLIENT_SECRET, ARM_TENANT_ID, ARM_SUBSCRIPTION_ID)
3. These files are centralised — shared across all projects, never inside a repo
