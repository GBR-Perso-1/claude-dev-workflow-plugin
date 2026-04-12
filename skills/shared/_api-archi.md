---
paths:
  - "api/**"
---

# API Architecture Rules

## Layer Structure
- Domain: entities, value objects, domain events. No dependencies.
- Application: commands, queries (CQRS), validators, interfaces. Depends on Domain only.
- Infrastructure: EF Core, external services, repositories. Implements Application interfaces.
- API: thin controllers (delegate to MediatR, no business logic), DTOs, middleware. Depends on Application.

## Dependency Rule
- Dependencies point inward only: API → Application → Domain
- Infrastructure implements interfaces defined in Application
- Never reference Infrastructure from Application or Domain

## Naming Conventions
- Commands: `{Verb}{Entity}Command`
- Queries: `Get{Entity}Query`, `List{Entities}Query`
- Handlers: `{Command/Query}Handler`
- Validators: `{Command/Query}Validator`
- Repositories: `I{Entity}Repository` (interface), `{Entity}Repository` (impl)

## Where Things Go
- New entity → Domain/Entities/
- New command/query → Application/Features/{Entity}/
- New validation → alongside its command in Application/Features/{Entity}/
- New controller endpoint → API/Controllers/{Entity}Controller.cs
- New DB config → Infrastructure/Persistence/Configurations/{Entity}Configuration.cs

## Handler Conventions
- Query naming: prefer `Get` prefix (`GetFooQuery` not `SearchFooQuery`)
- Controller actions: name after command/query, drop suffix (`CreateProperty` not `Create`)
- Query/Command + Handler in the same file — never split into separate files
- No cross-handler duplication — extract shared logic to services
- Complete one logical phase in `Handle` before starting the next
- `Handle` should read as a high-level sequence of steps; extract distinct logic into named private methods
