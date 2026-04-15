## Dev Settings Conventions

### Key naming convention

Keys follow the pattern: `<org>.<repo-slug>.<purpose>`

- `<org>` — the GitHub organisation or owner slug (e.g. `rise`)
- `<repo-slug>` — the repository name, kebab-cased (e.g. `wealth-pilot`)
- `<purpose>` — a short, descriptive identifier for the credential (e.g. `azure-client-secret`)

**Examples**

| Key | Assessment |
|-----|------------|
| `rise.wealth-pilot.azure-client-secret` | Good — org, repo, and purpose are all clear |
| `rise.wealth-pilot.db-connection-string` | Good — purpose unambiguous |
| `rise.auth-service.jwt-signing-key` | Good |
| `azure-secret` | Bad — missing org and repo; collides across projects |
| `wealth_pilot_secret` | Bad — underscores instead of dots; no org prefix |
| `WEALTH_PILOT_AZURE` | Bad — no structure; purpose unclear |

### Manifest format

Each Rise repo that requires injected dev settings must include a `.claude/dev-settings.json` at the repo root (committed to source control). It is a JSON array of entries, each with three required fields:

| Field | Type | Description |
|-------|------|-------------|
| `key` | string | The setting key — must match an entry in the private source (see naming convention above) |
| `targetFile` | string | Relative path from the repo root to the file that receives the value (e.g. `api/src/.env.local`) |
| `targetVariable` | string | Variable name to write inside that file (e.g. `AZURE_CLIENT_SECRET`) |

**Example**:
```json
[
  {
    "key": "rise.wealth-pilot.azure-client-secret",
    "targetFile": "api/src/.env.local",
    "targetVariable": "AZURE_CLIENT_SECRET"
  },
  {
    "key": "rise.wealth-pilot.db-password",
    "targetFile": "api/src/.env.local",
    "targetVariable": "DB_PASSWORD"
  }
]
```

The manifest contains no values — only mappings. Values are stored exclusively in the private source.

### Private source location

All credential values are stored in the private GitHub repository:

```
it--dev-settings
```

This repo is admin-controlled. Developers have read access via their existing `gh` CLI authentication. No additional credentials or tokens are required.

### Adding a new entry

To add a new setting key to the private source, contact the **platform admin (Guillaume)**. Provide:

1. The proposed key name (following the convention above).
2. The credential value.
3. A human-readable label describing what the credential is for.

Do not add entries to the private source repo yourself unless you have admin access.

### Manifest vs. private source

| File | Location | Contains | Committed to git? |
|------|----------|----------|-------------------|
| `.claude/dev-settings.json` | Each application repo | Key names, target files, target variables — **no values** | Yes |
| `settings.json` | `it--dev-settings` | Credential values | Private repo only |

The manifest (`.claude/dev-settings.json`) is public and reviewable — it is the contract between the repo and the injection skill. The private source contains only values and is never exposed in application repos.
