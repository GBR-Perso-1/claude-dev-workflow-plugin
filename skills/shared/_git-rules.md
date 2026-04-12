## Git rules for monorepo skills

- This is a **single monorepo** — all git commands run from the project root.
- Subprojects live under `api/`, `app/`, `infra/`.
- **NEVER** use compound commands like `cd path && git ...`.
- **NEVER** push to any remote without explicitly asking the user first.
- When scoping changes to a subproject, use path filters (e.g. `git diff -- api/`, `git add api/`).
