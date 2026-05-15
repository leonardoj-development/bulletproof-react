---
applyTo: "src/**/__tests__/**,src/**/*.test.{ts,tsx},e2e/**"
---

# Testing

## Testing pyramid (integration-first)

1. **Integration tests** — primary focus; test full feature workflows end-to-end within the app
2. **Unit tests** — shared utilities and complex pure logic only
3. **E2E tests** (Playwright) — critical user journeys (login, create/delete resource)

## Test setup

**Vitest config** in `vite.config.ts`:
```ts
test: {
  globals: true,
  environment: 'jsdom',
  setupFiles: './src/testing/setup-tests.ts',
  exclude: ['**/node_modules/**', '**/e2e/**'],
}
```

**Setup file** (`src/testing/setup-tests.ts`):
```ts
import { server } from './mocks/server';

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterAll(() => server.close());
beforeEach(() => {
  vi.stubGlobal('ResizeObserver', resizeObserverMock);
  initializeDb();
});
afterEach(() => {
  server.resetHandlers();
  resetDb();
});
```

`onUnhandledRequest: 'error'` ensures no real HTTP requests escape — every API call must have a handler.

## MSW: mock the server, not the function

Use MSW handlers instead of `vi.mock()` for API calls. This means components are tested as users experience them — with real data flow through React Query.

**Browser (dev mode):** `src/testing/mocks/browser.ts` — `setupWorker`
**Tests (Node):** `src/testing/mocks/server.ts` — `setupServer`

Handlers live in `src/testing/mocks/handlers/`:
- `auth.ts`, `discussions.ts`, `comments.ts`, `teams.ts`, `users.ts`
- All imported and combined in `handlers/index.ts`

## In-memory DB (`src/testing/mocks/db.ts`)

Uses `@mswjs/data` `factory()` to create a relational in-memory database. Handlers query and mutate this DB, not any real backend.

```ts
import { factory, primaryKey } from '@mswjs/data';
import { nanoid } from 'nanoid';

export const db = factory({
  user:       { id: primaryKey(nanoid), firstName: String, email: String, ... },
  team:       { id: primaryKey(nanoid), name: String, ... },
  discussion: { id: primaryKey(nanoid), title: String, authorId: String, teamId: String, ... },
  comment:    { id: primaryKey(nanoid), body: String, authorId: String, discussionId: String, ... },
});
```

Reset between tests via `initializeDb()` / `resetDb()` in setup.

## `renderApp` helper

Use `renderApp` (not `render`) for all integration tests. It:
1. Creates and logs in a test user
2. Mounts `<AppProvider>` with all real providers (React Query, Zustand, etc.)
3. Creates an in-memory router via `createMemoryRouter`
4. Waits for initial loading to finish

```ts
import { renderApp, screen, userEvent } from '@/testing/test-utils';

test('user can create a discussion', async () => {
  const user = await createUser();
  await renderApp(<DiscussionsRoute />, {
    user,
    path: '/app/discussions',
    url: '/app/discussions',
  });

  // Render unauthenticated: pass null
  // await renderApp(<SomeRoute />, { user: null });

  expect(screen.getByText(/discussions/i)).toBeInTheDocument();
});
```

**`renderApp` signature:**
```ts
renderApp(ui, {
  user?,    // undefined = create+login new user | AuthUser = use this | null = unauthenticated
  url?,     // initial URL (default: '/')
  path?,    // route path pattern (default: '/')
})
```

## Test utilities (`src/testing/test-utils.tsx`)

```ts
export { userEvent, rtlRender };
export * from '@testing-library/react'; // re-exports screen, waitFor, etc.

export const createUser = async (userProperties?: Partial<User>) => { ... };
export const loginAsUser = async (user) => { ... };
export const waitForLoadingToFinish = () =>
  waitForElementToBeRemoved(() => [
    ...screen.queryAllByTestId(/loading/i),
    ...screen.queryAllByText(/loading/i),
  ], { timeout: 4000 });
```

## Data generators (`src/testing/data-generators.ts`)

Create fake entities using `@ngneat/falso`:
```ts
export const createUser = (overrides?: Partial<User>): User => ({
  id: nanoid(),
  firstName: randFirstName(),
  email: randEmail(),
  role: 'USER',
  ...overrides,
});
```

## Integration test pattern

```ts
// src/app/routes/app/discussions/__tests__/discussion.test.tsx
import { renderApp, screen } from '@/testing/test-utils';

describe('Discussion', () => {
  it('should render the discussion details', async () => {
    const user = await createUser({ role: 'ADMIN' });
    const discussion = await createDiscussion({ authorId: user.id, teamId: user.teamId });

    await renderApp(<DiscussionRoute />, {
      user,
      url: `/app/discussions/${discussion.id}`,
      path: '/app/discussions/:discussionId',
    });

    expect(await screen.findByText(discussion.title)).toBeInTheDocument();
  });
});
```

## E2E tests (Playwright)

Located in `e2e/tests/`. Use `auth.setup.ts` to log in once and reuse the session:

```ts
// e2e/tests/auth.setup.ts
import { test as setup } from '@playwright/test';
setup('authenticate', async ({ page }) => {
  await page.goto('/auth/login');
  await page.fill('[name=email]', 'user@example.com');
  await page.fill('[name=password]', 'password');
  await page.click('[type=submit]');
  await page.waitForURL('/app');
  await page.context().storageState({ path: 'e2e/.auth/user.json' });
});
```

Run E2E: `yarn test-e2e` (starts mock server via `pm2` then runs Playwright).

## Rules

- Test behavior from the user's perspective — not implementation details
- Prefer `findBy*` (async) over `getBy*` for elements that appear after loading
- Never assert on component internals or call component methods directly
- Avoid `vi.mock()` for API calls — use MSW handlers instead
- Keep test files colocated with the route they test: `routes/app/discussions/__tests__/`
