---
applyTo: "src/**"
---

# State Management

## Decision tree: which state to use

```
Is the data fetched from a server?
  YES → React Query (TanStack Query)
  NO  → Is it needed across many unrelated components?
          YES → Zustand
          NO  → Is it complex with multiple interdependent updates?
                  YES → useReducer
                  NO  → useState
```

Never put server data into Zustand. Never cache server data manually — React Query owns it.

## React Query (server state)

**Default config** (`src/lib/react-query.ts`):

```ts
export const queryConfig = {
  queries: {
    refetchOnWindowFocus: false,
    retry: false,
    staleTime: 1000 * 60,  // 1 minute
  },
} satisfies DefaultOptions;
```

**QueryClient** is created once in `AppProvider` with `useState`:

```ts
const [queryClient] = React.useState(
  () => new QueryClient({ defaultOptions: queryConfig }),
);
```

**Key rules:**
- All data fetching goes through custom hooks in `features/<name>/api/`
- Use `queryOptions()` factory to keep query key and queryFn co-located
- Mutations always invalidate related queries via `queryClient.invalidateQueries()`
- Destructure `onSuccess` from `mutationConfig` to allow callers to hook into success without losing the invalidation

## Zustand (client/UI state)

Used only for global UI concerns: notifications, modals, themes. Not for server data.

**Pattern** — define store type inline:

```ts
// src/components/ui/notifications/notifications-store.ts
import { nanoid } from 'nanoid';
import { create } from 'zustand';

export type Notification = {
  id: string;
  type: 'info' | 'warning' | 'success' | 'error';
  title: string;
  message?: string;
};

type NotificationsStore = {
  notifications: Notification[];
  addNotification: (notification: Omit<Notification, 'id'>) => void;
  dismissNotification: (id: string) => void;
};

export const useNotifications = create<NotificationsStore>((set) => ({
  notifications: [],
  addNotification: (notification) =>
    set((state) => ({
      notifications: [...state.notifications, { id: nanoid(), ...notification }],
    })),
  dismissNotification: (id) =>
    set((state) => ({
      notifications: state.notifications.filter((n) => n.id !== id),
    })),
}));
```

Zustand stores can be accessed outside React components via `useNotifications.getState()` — used in the Axios error interceptor.

**Zustand mock for tests** (`__mocks__/zustand.ts`): reset all stores between tests automatically.

## Form state — React Hook Form + Zod

Always define the Zod schema before the form component:

```ts
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// 1. Schema (usually in the api/ file for reuse)
const createDiscussionInputSchema = z.object({
  title: z.string().min(1, 'Required'),
  body: z.string().min(1, 'Required'),
});
type CreateDiscussionInput = z.infer<typeof createDiscussionInputSchema>;

// 2. Form
const form = useForm<CreateDiscussionInput>({
  resolver: zodResolver(createDiscussionInputSchema),
});
```

Use the shared `<Form>` component from `src/components/ui/form/` which wraps `FormProvider` and handles `zodResolver` automatically — it accepts a `schema` prop and an `onSubmit` handler.

## State colocation principle

Keep state as close to where it is used as possible:

1. Start with `useState` inside the component
2. If two sibling components need it → lift to parent
3. If it spans unrelated parts of the app → Zustand
4. If it comes from the server → React Query

Avoid premature globalization. Do not put state in Zustand just because it "might be needed elsewhere later".
