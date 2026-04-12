---
paths:
  - "**/*.py"
  - "**/requirements*.txt"
  - "**/pyproject.toml"
---

# Python API Architecture Rules

## Architecture
- Clean Architecture: Api → Application → Domain ← Infrastructure
- Api: FastAPI endpoints, authentication, health checks
- Application: use cases / orchestration, each initialises its own services
- Domain: Pydantic models, enums, converters — no external dependencies
- Infrastructure: external service clients, repos, DB connections

## Resilience
- Retry logic on all external service calls (3 retries, exponential backoff `delay *= 2`)
- Error fallback methods: `_create_default_error_[type]` for graceful degradation
- Bulk validation: `len(results) == len(inputs)`, log `critical` on mismatch
