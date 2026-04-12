---
paths:
  - "app/**"
---

# Frontend App Architecture Rules

## State Management (Pinia)

- One store per domain concern (usePropertyStore, useTenantStore)
- Actions for async operations (API calls)
- Getters for computed/derived state
- Never mutate state outside actions

## API Integration

- API calls live in composables: `usePropertyApi()`
- Return { data, loading, error } pattern
- Base URL from env config, never hardcoded

## File Organization

- views/ — route-level pages
- components/ — reusable UI components
- composables/ — shared logic (useXxx)
- stores/ — Pinia stores
- types/ — TypeScript interfaces/types
- api/ — API client wrappers

## Component & Composable Placement

- Code starts close to where it's used and migrates outward as reuse grows:
  1. **Page-scoped** — lives next to the page that uses it
  2. **Feature-scoped** — shared by nearby pages, move to nearest common ancestor
  3. **App-scoped** (`appComponents/`, `composables/`) — used across feature paths
  4. **Generic** (`components/` root: `inputs/`, `displays/`, `dialogs/`) — truly reusable, no business logic

## Safety

- **Never touch `AuthHelper.ts`** — MSAL authentication wrapper, changes break auth across the app

## Service Layer

- Wrap auto-generated API clients via a service handler — never call generated clients directly
- Services export async functions that delegate to the handler: `performApiFetch()`, `performApiAction()`

## Store Patterns

- Use factory functions for query-string sync stores and refresh-trigger stores
- Never hand-roll a store when a factory covers the need

## Navigation

- Navigation is handled via `.project/navigation.json`
- When `app/src/router/RoutePages.ts` changes, you likely need to update `navigation.json` too
- This file is uploaded manually — do not modify the `navigationItem` table in the API
