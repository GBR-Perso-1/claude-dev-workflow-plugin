---
paths:
  - "**/*.cs"
  - "**/appsettings*.json"
  - "**/*.csproj"
---

# .NET / C# Language Rules

## Controllers

- Return ActionResult<T> with proper HTTP status codes
- Use [ApiController] attribute

## CQRS with MediatR

- Commands mutate state, return created/updated entity ID
- Queries are read-only, return DTOs (never entities)
- FluentValidation validator per command for input validation

## Entity Framework

- DbContext in Infrastructure layer
- EntityTypeConfiguration per entity (no data annotations on domain)
- Migrations via `dotnet ef migrations add {Name}`
- SQL script generation: use skill `/repo-ef-sql`
- Always seed lookup/reference data in configurations
- Centralized NuGet versions via `Directory.Packages.props`

## DTOs

- Request DTOs: match command properties
- Response DTOs: flat, no nested entities unless necessary

## Error Handling

- Domain exceptions for business rule violations
- Middleware catches and maps to ProblemDetails
- Never expose stack traces in responses

## C# Style

- Never use `var` — always explicit types
- No `Async` suffix on method names — everything is async, the suffix adds no information
- Collection expressions: `[]` not `new List<T>()`; prefer `[.. enumerable]` over `.ToList()`
- Bulk deletes: `.Where(...).ExecuteDeleteAsync(cancellationToken)` — never load entities to delete them
- No `#region` directives
- Prefer ternary/conditional over `if` for return/assign — format as 3 lines (condition, `? true`, `: false`)
- Records for commands, queries, DTOs, and value objects; sealed classes for handlers
- `ProducesResponseType` attributes on all controller actions
- Nullable reference types enabled, implicit usings enabled
- .NET 10 / C# 14 features valid (primary constructors, collection expressions)

## C# Naming & Formatting

- PascalCase for constants, `s_` prefix for static fields, `_camelCase` for private fields
- 4-space indentation, braces on new lines, 120-char line guideline
- Keep methods under 30 lines

## Safety

- Never generate OpenAPI specs — these are managed at build time
- Ask before modifying Program.cs, any migration, or any Domain entity
- Never rename public API routes without flagging it

## Query Projections

- When a query returns a DTO needing related data, add navigation properties and create `{Feature}QueryExtensions.cs` in `Features/{Feature}/Extensions/`
- Define `ProjectToDto()` extension on `IQueryable<TEntity>` that projects to DTO inside `Select()` — single SQL query, no in-memory transformation
- Both list and single-item handlers share the same projection extension
