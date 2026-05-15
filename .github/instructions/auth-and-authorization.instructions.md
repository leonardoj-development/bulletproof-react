---
applyTo: "src/lib/auth.tsx,src/lib/authorization.tsx,src/features/auth/**"
---

# Authentication & Authorization

## Authentication: `react-query-auth` + JWT cookies

Auth state is managed by `react-query-auth` configured in `src/lib/auth.tsx`. It wraps TanStack Query and exposes hooks + a loader component.

**Configuration:**
```ts
// src/lib/auth.tsx
import { configureAuth } from 'react-query-auth';

const authConfig = {
  userFn: getUser,          // GET /auth/me — returns current user or throws
  loginFn: async (data) => {
    const response = await loginWithEmailAndPassword(data);
    return response.user;
  },
  registerFn: async (data) => {
    const response = await registerWithEmailAndPassword(data);
    return response.user;
  },
  logoutFn: logout,         // POST /auth/logout
};

export const { useUser, useLogin, useLogout, useRegister, AuthLoader } =
  configureAuth(authConfig);
```

**Hooks:**
- `useUser()` — returns `{ data: User | null, ... }`; null = unauthenticated
- `useLogin()` — mutation hook; call `.mutate({ email, password })`
- `useRegister()` — mutation hook
- `useLogout()` — mutation hook

**`AuthLoader`** wraps the app and shows a spinner while the initial user fetch resolves:
```tsx
// src/app/provider.tsx
<AuthLoader renderLoading={() => <Spinner />}>
  {children}
</AuthLoader>
```

## Zod schemas for auth inputs

```ts
export const loginInputSchema = z.object({
  email: z.string().min(1, 'Required').email('Invalid email'),
  password: z.string().min(5, 'Required'),
});

export const registerInputSchema = z
  .object({
    email: z.string().min(1, 'Required'),
    firstName: z.string().min(1, 'Required'),
    lastName: z.string().min(1, 'Required'),
    password: z.string().min(5, 'Required'),
  })
  .and(
    // Must join an existing team OR create a new one — not both
    z.object({ teamId: z.string().min(1, 'Required'), teamName: z.null().default(null) })
     .or(z.object({ teamName: z.string().min(1, 'Required'), teamId: z.null().default(null) })),
  );
```

## Token storage

JWTs are stored in **HttpOnly cookies** (set by the server). The Axios instance sends `withCredentials: true` on every request, so cookies are automatically included. The client never reads the token directly.

## Protected routes

`ProtectedRoute` redirects unauthenticated users to login, preserving the intended URL in `?redirectTo`:

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

Wrap protected route trees in `router.tsx`, not individual pages.

## Authorization: RBAC + PBAC (`src/lib/authorization.tsx`)

Two complementary models:

### RBAC — Role-Based Access Control

```ts
export enum ROLES {
  ADMIN = 'ADMIN',
  USER = 'USER',
}
```

`ADMIN` = team creator; `USER` = team member. Roles are stored on the `User` entity.

**Hook:**
```ts
const { checkAccess, role } = useAuthorization();
const canCreate = checkAccess({ allowedRoles: [ROLES.ADMIN] });
```

### PBAC — Permission-Based Access Control

Defined as a `POLICIES` map. Each key is a permission string; each value is a function `(user, resource) => boolean`:

```ts
export const POLICIES = {
  'comment:delete': (user: User, comment: Comment) => {
    if (user.role === 'ADMIN') return true;
    if (user.role === 'USER' && comment.author?.id === user.id) return true;
    return false;
  },
};
```

Evaluate a policy in a component:
```ts
const user = useUser();
const canDelete = POLICIES['comment:delete'](user.data!, comment);
```

### `<Authorization>` component

Gate UI rendering with the `Authorization` wrapper. Use `allowedRoles` for RBAC or `policyCheck` for PBAC — never both at once:

```tsx
// RBAC — only ADMIN sees the create button
<Authorization allowedRoles={[ROLES.ADMIN]}>
  <CreateDiscussion />
</Authorization>

// PBAC — only comment author or admin can delete
<Authorization
  policyCheck={POLICIES['comment:delete'](user.data!, comment)}
  forbiddenFallback={<span>No permission</span>}
>
  <DeleteComment commentId={comment.id} />
</Authorization>
```

**Rules:**
- Client-side authorization is for UX only — always enforce on the server
- `forbiddenFallback` defaults to `null` (renders nothing when access denied)
- Never bypass `<Authorization>` for permissions logic — don't use ternaries directly
