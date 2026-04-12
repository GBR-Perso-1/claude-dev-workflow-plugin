---
paths:
  - "**/*.vue"
  - "**/*.ts"
  - "**/*.tsx"
  - "**/package.json"
---

# Vue / TypeScript Language Rules

## Component Structure
- Composition API only (no Options API)
- Script setup syntax: `<script setup lang="ts">`
- One component per file
- Props defined inline with `defineProps<{ ... }>()` — no separate interface

## Naming
- Components: PascalCase (`PropertyCard.vue`), PascalCase in templates too (`<PageLayout>` not `<page-layout>`)
- Composables: camelCase with `use` prefix (`usePropertyApi.ts`)
- Stores: camelCase with `use` prefix (`usePropertyStore.ts`)
- Types: PascalCase (`PropertyDto.ts`), `I` prefix for DTO interfaces (`IPropertyDto`)

## Code Style
- Always run Prettier on modified files immediately after editing
- Axios only — never `fetch()`
- Never use `any` in TypeScript
- Quasar components preferred over custom CSS
- Never add raw CSS outside the SCSS files — use Quasar utilities or the SCSS files
- Split components at 200–300 lines — extract to `useXxx.ts` composable
- Constant arrow functions over function declarations
- CSS variables (`--q-primary`, etc.) not hardcoded colours — dark mode depends on it
- Never use raw `rgba()` — define a palette variable in `:root` first, then reference via `var()`
- `<prefix>-styles.scss` = shared org/project design system (e.g. `ekla-styles.scss` — prefix matches the project prefix); `app-styles.scss` = app-specific only
