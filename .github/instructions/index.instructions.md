---
applyTo: "**"
description: "Use when starting a new feature, exploring the architecture, or unsure which instruction file to consult. Index of all Bulletproof React AI specs."
---

# Bulletproof React — AI Specs Index

Architecture instructions for generating projects following the Bulletproof React patterns.
Each spec is atomic and focused on a single concern.

## Cómo se cargan estas specs

Cada spec tiene un `applyTo` en su frontmatter: un glob pattern que le indica a Copilot qué archivos del proyecto activan esa spec **automáticamente**. Cuando el archivo que estás editando coincide con el patrón, Copilot incluye esa spec en su contexto sin que tengas que pedirlo.

Este archivo (`index`) no tiene `applyTo` — se carga on-demand cuando Copilot detecta que la tarea requiere orientación arquitectural general (gracias al campo `description`), o al adjuntarlo manualmente con "Add Context → Instructions".

## Specs

| File | Covers | Se carga al editar... |
|------|--------|-----------------------|
| [tech-stack.instructions.md](tech-stack.instructions.md) | Libraries, versions, and their role in the stack | cualquier archivo (`**`) |
| [project-structure.instructions.md](project-structure.instructions.md) | Folder layout, feature modules, naming conventions | cualquier archivo en `src/` |
| [api-layer.instructions.md](api-layer.instructions.md) | Fetchers, queryOptions, React Query hooks, Axios client | `src/features/**/api/**`, `src/lib/api-client.ts` |
| [state-management.instructions.md](state-management.instructions.md) | React Query, Zustand, React Hook Form + Zod, colocation | cualquier archivo en `src/` |
| [components.instructions.md](components.instructions.md) | CVA variants, Radix UI, Tailwind, composition patterns | `src/components/**`, `src/features/**/components/**` |
| [routing.instructions.md](routing.instructions.md) | React Router v7, protected routes | `src/app/router.tsx`, `src/app/routes/**`, `src/config/paths.ts` |
| [testing.instructions.md](testing.instructions.md) | Vitest, MSW, `renderApp`, integration test patterns | `src/**/__tests__/**`, `src/**/*.test.{ts,tsx}`, `e2e/**` |
| [auth-and-authorization.instructions.md](auth-and-authorization.instructions.md) | JWT cookies, react-query-auth, RBAC, PBAC | `src/lib/auth.tsx`, `src/lib/authorization.tsx`, `src/features/auth/**` |
| [error-handling.instructions.md](error-handling.instructions.md) | Axios interceptor, Error Boundaries, toast notifications | `src/components/errors/**`, `src/app/provider.tsx`, `src/lib/api-client.ts` |

## Architecture in one glance

```
src/
├── app/          → router.tsx + provider.tsx + routes/   [routing]
├── components/   → ui/, layouts/, errors/                [components]
├── config/       → env.ts + paths.ts                     [project-structure]
├── features/
│   └── <name>/
│       ├── api/        → fetcher + queryOptions + hook   [api-layer]
│       └── components/ → feature UI                      [components]
├── lib/          → api-client, auth, authorization,       [auth, error-handling]
│                   react-query
├── testing/      → test-utils, mocks/, data-generators    [testing]
├── types/        → API response shapes
└── utils/        → cn(), format()
```

## Data flow

```
Zod schema → React Hook Form → useMutation
                                    ↓
                          api (Axios) → MSW handler → @mswjs/data
                                    ↓
                          invalidateQueries → useQuery re-fetch → UI
```

## Key decisions at a glance

- **Server data** → always React Query, never Zustand
- **Global UI state** → Zustand (`useNotifications`, modals)
- **Form state** → React Hook Form + `zodResolver`
- **HTTP errors** → handled globally in Axios interceptor (no try/catch in components)
- **Authorization** → `<Authorization allowedRoles>` (RBAC) or `policyCheck` (PBAC)
- **Tests** → `renderApp` + MSW, never `vi.mock()` for HTTP
- **Routing** → all paths from `src/config/paths.ts`, all routes lazy-loaded
- **Components** → copied into `src/components/ui/` (Shadcn pattern), styled with `cn()` + CVA
