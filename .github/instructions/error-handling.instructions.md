---
applyTo: "src/components/errors/**,src/app/provider.tsx,src/app/routes/**/root.tsx,src/lib/api-client.ts"
---

# Error Handling

## Three layers of error handling

| Layer | Mechanism | Scope |
|-------|-----------|-------|
| API / HTTP | Axios response interceptor | All HTTP errors (network, 4xx, 5xx) |
| React render | Error Boundaries | Component tree crashes |
| Async data | React Query error state | Failed queries / mutations |

## 1. Axios response interceptor (`src/lib/api-client.ts`)

Handles all HTTP errors globally. Components do not need try/catch for API calls:

```ts
api.interceptors.response.use(
  (response) => response.data,  // unwrap data on success
  (error) => {
    const message = error.response?.data?.message || error.message;

    // Show toast notification for every error
    useNotifications.getState().addNotification({
      type: 'error',
      title: 'Error',
      message,
    });

    // Redirect to login on 401 (expired/missing session)
    if (error.response?.status === 401) {
      const redirectTo = window.location.pathname;
      window.location.href = paths.auth.login.getHref(redirectTo);
    }

    return Promise.reject(error);
  },
);
```

Note: `useNotifications.getState()` accesses the Zustand store outside React — valid pattern for interceptors.

## 2. Error Boundaries

Place error boundaries at **multiple levels** — not just the app root:

### App-level (`src/app/provider.tsx`)
Catches any unrecoverable render error across the whole app:

```tsx
<ErrorBoundary FallbackComponent={MainErrorFallback}>
  {/* entire app */}
</ErrorBoundary>
```

```tsx
// src/components/errors/main.tsx
export const MainErrorFallback = () => (
  <div className="flex h-screen w-screen flex-col items-center justify-center text-red-500" role="alert">
    <h2 className="text-lg font-semibold">Ooops, something went wrong :(</h2>
    <Button onClick={() => window.location.assign(window.location.origin)}>
      Refresh
    </Button>
  </div>
);
```

### Route-level (`src/app/routes/app/root.tsx`)
Catches errors within the protected app shell without crashing the whole page:

```ts
// root.tsx
export const ErrorBoundary = () => {
  return <div>Something went wrong!</div>;
};
```

In React Router, exporting `ErrorBoundary` from a route file automatically attaches it to that route segment.

### Feature-level
For features with complex data dependencies, add an `ErrorBoundary` around the feature component:

```tsx
import { ErrorBoundary } from 'react-error-boundary';

<ErrorBoundary fallback={<div>Could not load discussions.</div>}>
  <DiscussionsList />
</ErrorBoundary>
```

## 3. React Query error state

Mutations surface errors through `mutation.error` or via the global interceptor toast. Queries surface errors via `query.error`:

```tsx
const discussionsQuery = useDiscussions({ page });

if (discussionsQuery.isError) {
  return <div>Failed to load discussions.</div>;
}
```

`retry: false` is set in global query config — errors appear immediately without automatic retries.

## Notifications (toast system)

All error notifications go through the Zustand notifications store:

```ts
const { addNotification } = useNotifications();

addNotification({ type: 'error', title: 'Delete failed', message: error.message });
addNotification({ type: 'success', title: 'Discussion created' });
```

The `<Notifications />` component in `AppProvider` renders all active toasts. Never render toasts directly in feature components.

## Rules

- Do not wrap every component in try/catch — let the interceptor and error boundaries handle it
- Place error boundaries at route level at minimum; add feature-level boundaries for isolated widgets
- Use `role="alert"` on error fallback UI for accessibility
- Always provide a recovery action in error UIs (e.g., Refresh button, Try again link)
