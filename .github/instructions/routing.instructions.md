---
applyTo: "src/app/router.tsx,src/app/routes/**,src/config/paths.ts"
---

# Routing

## Router: React Router v7 `createBrowserRouter` — `src/app/router.tsx`

## URL structure (all three)

```
/                          → Landing page
/auth/login                → Login
/auth/register             → Register
/app                       → Dashboard (protected)
/app/discussions           → Discussions list (protected)
/app/discussions/:id       → Discussion detail (protected)
/app/users                 → Team members (protected)
/app/profile               → User profile (protected)
```

## Path constants (`src/config/paths.ts`)

All URL strings live here. Never hardcode paths in components or routers.

```ts
export const paths = {
  home: { path: '/', getHref: () => '/' },
  auth: {
    login: {
      path: '/auth/login',
      getHref: (redirectTo?: string | null) =>
        `/auth/login${redirectTo ? `?redirectTo=${encodeURIComponent(redirectTo)}` : ''}`,
    },
    register: { path: '/auth/register', getHref: () => '/auth/register' },
  },
  app: {
    root:        { path: '/app',                      getHref: () => '/app' },
    dashboard:   { path: '',                           getHref: () => '/app' },
    discussions: { path: 'discussions',                getHref: () => '/app/discussions' },
    discussion:  { path: 'discussions/:discussionId',  getHref: (id: string) => `/app/discussions/${id}` },
    users:       { path: 'users',                      getHref: () => '/app/users' },
    profile:     { path: 'profile',                    getHref: () => '/app/profile' },
  },
} as const;
```

## react-vite: `createAppRouter`

```ts
// src/app/router.tsx
const convert = (queryClient: QueryClient) => (m: any) => {
  const { clientLoader, clientAction, default: Component, ...rest } = m;
  return {
    ...rest,
    loader: clientLoader?.(queryClient),
    action: clientAction?.(queryClient),
    Component,
  };
};

export const createAppRouter = (queryClient: QueryClient) =>
  createBrowserRouter([
    {
      path: paths.home.path,
      lazy: () => import('./routes/landing').then(convert(queryClient)),
    },
    {
      path: paths.app.root.path,
      element: <ProtectedRoute><AppRoot /></ProtectedRoute>,
      ErrorBoundary: AppRootErrorBoundary,
      children: [
        {
          path: paths.app.discussions.path,
          lazy: () => import('./routes/app/discussions/discussions').then(convert(queryClient)),
        },
        // ... other children
      ],
    },
    { path: '*', lazy: () => import('./routes/not-found').then(convert(queryClient)) },
  ]);
```

**Rules:**
- All routes are **lazy-loaded** with `lazy` + dynamic `import()`
- Protected routes are wrapped in `<ProtectedRoute>` (redirects to login if no user)
- `clientLoader` is a function that accepts `queryClient` and returns the React Router loader
- `AppRoot` renders `<DashboardLayout><Outlet /></DashboardLayout>`

## Route file pattern (react-vite)

Each route file exports a default component and optionally a `clientLoader`:

```ts
// src/app/routes/app/discussions/discussions.tsx
import { QueryClient } from '@tanstack/react-query';
import { getDiscussionsQueryOptions } from '@/features/discussions/api/get-discussions';

// Optional: prefetch data before component mounts
export const clientLoader = (queryClient: QueryClient) => async () => {
  return queryClient.ensureQueryData(getDiscussionsQueryOptions());
};

const DiscussionsRoute = () => {
  return <DiscussionsList />;
};

export default DiscussionsRoute;
```

## Protected routes

In react-vite, `ProtectedRoute` lives in `src/lib/auth.tsx`:

```ts
export const ProtectedRoute = ({ children }: { children: React.ReactNode }) => {
  const user = useUser();
  const location = useLocation();

  if (!user.data) {
    return <Navigate to={paths.auth.login.getHref(location.pathname)} replace />;
  }
  return children;
};
```


