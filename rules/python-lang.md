---
paths:
  - "**/*.py"
  - "**/requirements*.txt"
  - "**/pyproject.toml"
---

# Python Language Rules

## Pydantic Models
- `Field(description=...)` on all fields
- `ConfigDict(extra="forbid")` — strict, no extra fields allowed
- Enums always include `Unknown` / `Other` fallbacks

## Type Safety
- Type hints required on all function parameters and return types
- Never use `Any` unless interfacing with untyped third-party code

## Naming
- Classes: `PascalCase`
- Functions / variables: `snake_case`
- Constants: `UPPER_CASE`
- Pydantic field names: `camelCase` (for JSON serialisation)

## Imports
- Standard library → third-party → local (e.g. `src.Domain.Models...`)

## Logging
- `info` for progress, `warning` for retries, `error` for failures (with `exc_info=True`), `critical` for data integrity issues

## Formatting
- Ruff formatter and linter
- Max line length: 120
- Keep functions under 30 lines
