---
applyTo: "**"
---

# Tech Stack

## Core
- **React 18.3** — concurrent features, Suspense, transitions
- **TypeScript** — strict mode enabled; absolute imports via `@/` prefix
- **Vite 5**

## Routing
- **react-router v7** — `createBrowserRouter` + lazy routes

## Server State
- **@tanstack/react-query ^5.32** — all remote data; `queryOptions`, `useQuery`, `useMutation`
- **react-query-auth ^2.4** — authentication state built on top of React Query

## Client State
- **zustand ^4.5** — global UI state (notifications, modals); accessed via `create<Store>()`

## Forms & Validation
- **react-hook-form ^7.51** — form management
- **zod ^3.23** — schema validation; always define schema before the fetcher
- **@hookform/resolvers ^3.3** — `zodResolver` bridge

## HTTP
- **axios ^1.6.8** — single configured instance in `src/lib/api-client.ts`
- **msw ^2.2** — Mock Service Worker; used in both dev (browser) and tests (Node)
- **@mswjs/data ^0.16** — in-memory relational DB for MSW handlers

## UI & Styling
- **Tailwind CSS** — utility-first; configured in `tailwind.config.cjs`
- **Radix UI primitives** — headless, unstyled; imported individually:
  - `@radix-ui/react-dialog`, `@radix-ui/react-dropdown-menu`, `@radix-ui/react-label`, `@radix-ui/react-slot`, `@radix-ui/react-switch`
- **class-variance-authority ^0.7** — `cva()` for component variants
- **clsx ^2.1** + **tailwind-merge ^2.3** — combined via `cn()` utility
- **lucide-react ^0.378** — icons
- **tailwindcss-animate ^1.0** — animation utilities

## Testing
- **vitest** — test runner (Jest-compatible API); configured in `vite.config.ts`
- **@testing-library/react ^15** — component rendering and queries
- **@testing-library/user-event ^14** — realistic user interaction simulation
- **@playwright/test ^1.49** — E2E tests under `e2e/tests/`

## Development Tools
- **Storybook 8** — component development (`yarn storybook`)
- **Plop** — code generation templates (`yarn generate`)
- **Husky** — git hooks (pre-commit: lint + type-check; pre-push: tests)
- **ESLint 8** + **Prettier** — linting and formatting
- **eslint-plugin-check-file** — enforces kebab-case file naming

## Utilities
- **dayjs ^1.11** — date formatting
- **nanoid ^5.0** — unique ID generation
- **dompurify ^3.1** — XSS prevention
- **marked ^12** — Markdown parsing
- **@ngneat/falso ^7.2** — fake data generation for tests and MSW seeds

## Environment Variables
Variables are validated at startup with Zod in `src/config/env.ts`.
In Vite apps, variables must be prefixed `VITE_APP_` and are stripped of that prefix internally.

```ts
// src/config/env.ts pattern
const EnvSchema = z.object({
  API_URL: z.string(),
  ENABLE_API_MOCKING: z.string().transform(s => s === 'true').optional(),
  APP_URL: z.string().optional().default('http://localhost:3000'),
});
```
