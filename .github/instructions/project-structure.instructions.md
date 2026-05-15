---
applyTo: "src/**"
---

# Project Structure

## Top-level `src/` layout

```
src/
├── app/          # Application layer: providers, router, route components
├── components/   # Shared UI components (used across features)
├── config/       # Environment variables and route path constants
├── features/     # Feature modules — the majority of the business logic
├── hooks/        # Shared React hooks (used across features)
├── lib/          # Preconfigured third-party library instances
├── testing/      # Test utilities, MSW mocks, data generators
├── types/        # Shared TypeScript types (API response shapes)
└── utils/        # Shared utility functions (pure, no React)
```

## Dependency flow (unidirectional — never reverse)

```
utils / types / config
       ↓
      lib
       ↓
  components / hooks
       ↓
    features
       ↓
      app
```

**Rules:**
- Features must NOT import from other features.
- `app/` may import from features; features must NOT import from `app/`.
- Shared code (hooks, components, utils) must NOT import from features.

## Feature module structure

Each feature is self-contained under `src/features/<feature-name>/`:

```
src/features/discussions/
├── api/         # Fetchers, queryOptions, React Query hooks
├── components/  # Components used only within this feature
├── hooks/       # Hooks used only within this feature
├── stores/      # Zustand stores scoped to this feature (if needed)
├── types/       # TypeScript types for this feature (if not in global types/)
└── utils/       # Pure utilities scoped to this feature
```

Only `api/` and `components/` are required; other subdirs are created on demand.

## File naming conventions

| Type | Convention | Example |
|------|-----------|---------|
| React components | `kebab-case.tsx` | `discussion-view.tsx` |
| Hooks | `use-kebab-case.ts` | `use-discussions.ts` |
| Utilities | `kebab-case.ts` | `format-date.ts` |
| API files | `verb-noun.ts` | `get-discussions.ts`, `create-discussion.ts` |
| Types | `kebab-case.ts` | `api-types.ts` |
| Folders | `kebab-case` | `discussions/`, `api/`, `user-management/` |
| Test files | `*.test.tsx` inside `__tests__/` | `discussion.test.tsx` |

## `app/` directory layout (react-vite)

```
src/app/
├── index.tsx           # Root <App /> component
├── provider.tsx        # Wraps the whole app with all providers
├── router.tsx          # createBrowserRouter config with lazy routes
└── routes/
    ├── landing.tsx
    ├── not-found.tsx
    ├── auth/
    │   ├── login.tsx
    │   └── register.tsx
    └── app/
        ├── root.tsx            # Protected layout shell (Outlet)
        ├── dashboard.tsx
        ├── profile.tsx
        ├── users.tsx
        └── discussions/
            ├── discussions.tsx
            ├── discussion.tsx
            └── __tests__/
                └── discussion.test.tsx
```

## `components/ui/` structure (shared)

Each UI component lives in its own folder with an `index.ts` barrel:

```
components/ui/
├── button/
│   ├── button.tsx
│   ├── button.stories.tsx
│   └── index.ts
├── form/
├── dialog/
├── drawer/
├── notifications/
├── spinner/
├── table/
└── ...
```

## `config/` contents

- `env.ts` — Zod-validated environment variables; the only place to access `import.meta.env` / `process.env`
- `paths.ts` — Typed route path constants with `getHref()` helpers; the only place to define URL strings

## `lib/` contents

- `api-client.ts` — Single Axios instance with auth and error interceptors
- `auth.tsx` — `configureAuth` from react-query-auth + `ProtectedRoute`
- `authorization.tsx` — `ROLES` enum, `POLICIES` map, `useAuthorization`, `<Authorization>`
- `react-query.ts` — `queryConfig` defaults + `MutationConfig` / `QueryConfig` type helpers
