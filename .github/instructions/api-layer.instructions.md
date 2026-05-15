---
applyTo: "src/features/**/api/**,src/lib/api-client.ts"
---

# API Layer

## Pattern: every API file has 3 exports

Each file under `src/features/<feature>/api/` exports exactly these three things:

1. **Fetcher function** — plain async function, calls `api` (Axios instance)
2. **`queryOptions` / mutation options factory** — TanStack Query options object
3. **Custom hook** — wraps the query/mutation; consumed by components

```ts
// src/features/discussions/api/get-discussions.ts
import { queryOptions, useQuery } from '@tanstack/react-query';
import { api } from '@/lib/api-client';
import { QueryConfig } from '@/lib/react-query';
import { Discussion, Meta } from '@/types/api';

// 1. Fetcher
export const getDiscussions = (page = 1): Promise<{ data: Discussion[]; meta: Meta }> => {
  return api.get('/discussions', { params: { page } });
};

// 2. Query options factory (reusable — used by loaders, prefetch, and the hook)
export const getDiscussionsQueryOptions = ({ page }: { page?: number } = {}) => {
  return queryOptions({
    queryKey: page ? ['discussions', { page }] : ['discussions'],
    queryFn: () => getDiscussions(page),
  });
};

// 3. Custom hook
type UseDiscussionsOptions = {
  page?: number;
  queryConfig?: QueryConfig<typeof getDiscussionsQueryOptions>;
};

export const useDiscussions = ({ queryConfig, page }: UseDiscussionsOptions = {}) => {
  return useQuery({
    ...getDiscussionsQueryOptions({ page }),
    ...queryConfig,
  });
};
```

## Pattern: mutations (create / update / delete)

Define the Zod schema first. Invalidate related queries on success. Accept `mutationConfig` for external side-effects.

```ts
// src/features/discussions/api/create-discussion.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { z } from 'zod';
import { api } from '@/lib/api-client';
import { MutationConfig } from '@/lib/react-query';
import { getDiscussionsQueryOptions } from './get-discussions';

// Schema first — before any function
export const createDiscussionInputSchema = z.object({
  title: z.string().min(1, 'Required'),
  body: z.string().min(1, 'Required'),
});
export type CreateDiscussionInput = z.infer<typeof createDiscussionInputSchema>;

// Fetcher
export const createDiscussion = ({ data }: { data: CreateDiscussionInput }): Promise<Discussion> => {
  return api.post('/discussions', data);
};

// Hook
type UseCreateDiscussionOptions = {
  mutationConfig?: MutationConfig<typeof createDiscussion>;
};

export const useCreateDiscussion = ({ mutationConfig }: UseCreateDiscussionOptions = {}) => {
  const queryClient = useQueryClient();
  const { onSuccess, ...restConfig } = mutationConfig || {};

  return useMutation({
    onSuccess: (...args) => {
      // Always invalidate the related list query
      queryClient.invalidateQueries({ queryKey: getDiscussionsQueryOptions().queryKey });
      onSuccess?.(...args);
    },
    ...restConfig,
    mutationFn: createDiscussion,
  });
};
```

## Axios instance (`src/lib/api-client.ts`)

- Single `api` instance; `baseURL` from `env.API_URL`
- Request interceptor sets `Accept: application/json` and `withCredentials: true`
- Response interceptor:
  - Unwraps `response.data` on success
  - Shows toast notification on error
  - Redirects to login on 401

Never create additional Axios instances. Always import `api` from `@/lib/api-client`.

## Type helpers (`src/lib/react-query.ts`)

```ts
// Make query config optional params type-safe
export type QueryConfig<T extends (...args: any[]) => any> = Omit<
  ReturnType<T>,
  'queryKey' | 'queryFn'
>;

// Type mutations against the fetcher function signature
export type MutationConfig<MutationFnType extends (...args: any) => Promise<any>> =
  UseMutationOptions<
    ApiFnReturnType<MutationFnType>,
    Error,
    Parameters<MutationFnType>[0]
  >;
```

## Query key conventions

- Use noun arrays: `['discussions']`, `['discussions', { page }]`, `['discussion', id]`
- List queries use the entity name alone or with filter params
- Detail queries include the entity ID: `['discussion', discussionId]`
- Always derive the key from the `queryOptions` factory — never hardcode it in `invalidateQueries`

## Data prefetching (react-vite with React Router loaders)

Expose a `clientLoader` from the route file to prefetch before component mounts:

```ts
// src/app/routes/app/discussions/discussions.tsx
export const clientLoader = (queryClient: QueryClient) => async () => {
  const query = getDiscussionsQueryOptions();
  return queryClient.ensureQueryData(query);
};
```

For on-hover prefetch inside a component:
```ts
queryClient.prefetchQuery(getDiscussionQueryOptions(id));
```

## Next.js App Router differences

Use `HydrationBoundary` to pass server-prefetched data to the client:

```ts
// Server component (page.tsx)
const queryClient = new QueryClient();
await queryClient.prefetchQuery(getDiscussionsQueryOptions());
const dehydratedState = dehydrate(queryClient);

return (
  <HydrationBoundary state={dehydratedState}>
    <DiscussionsList />
  </HydrationBoundary>
);
```
