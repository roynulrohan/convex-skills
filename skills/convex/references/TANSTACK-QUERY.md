# TanStack Query Integration

Use Convex with TanStack React Query via `@convex-dev/react-query`. Convex's real-time subscriptions push data into TanStack Query's cache automatically — no polling, no manual invalidation.

> For core Convex React hooks, see `convex/references/CLIENT.md`.
> For a comparison of query approaches, see the "Query Approaches Comparison" table in CLIENT.md.

## Installation

```bash
npm install @convex-dev/react-query @tanstack/react-query
```

Peer dependencies: `@tanstack/react-query` ^5.0.0, `convex` ^1.31.7

## Provider Setup

Three objects must be wired together: `ConvexReactClient`, `ConvexQueryClient`, and TanStack `QueryClient`.

```typescript
import { ConvexReactClient, ConvexProvider } from 'convex/react';
import { ConvexQueryClient } from '@convex-dev/react-query';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const convexClient = new ConvexReactClient(import.meta.env.VITE_CONVEX_URL);
const convexQueryClient = new ConvexQueryClient(convexClient);
const queryClient = new QueryClient({
    defaultOptions: {
        queries: {
            queryKeyHashFn: convexQueryClient.hashFn(),
            queryFn: convexQueryClient.queryFn(),
        },
    },
});
convexQueryClient.connect(queryClient);

function App() {
    return (
        <ConvexProvider client={convexClient}>
            <QueryClientProvider client={queryClient}>
                <YourApp />
            </QueryClientProvider>
        </ConvexProvider>
    );
}
```

**`hashFn()` and `queryFn()` accept fallbacks** for non-Convex queries:

```typescript
queryKeyHashFn: convexQueryClient.hashFn(existingHashFn),
queryFn: convexQueryClient.queryFn(existingQueryFn),
```

Without fallbacks, non-Convex query keys will throw.

## Reactive Queries with `convexQuery`

The primary API. Returns options to spread into TanStack Query's `useQuery()`:

```typescript
import { useQuery } from '@tanstack/react-query';
import { convexQuery } from '@convex-dev/react-query';
import { api } from '../convex/_generated/api';

function MessageList({ channelId }: { channelId: string }) {
    const { data, isPending, error } = useQuery({
        ...convexQuery(api.messages.list, { channelId }),
        gcTime: 10000, // keep subscription alive 10s after unmount
    });

    if (isPending) return <Spinner />;
    if (error) return <Error message={error.message} />;

    return data.map((msg) => <Message key={msg._id} message={msg} />);
}
```

`convexQuery()` returns:

- `queryKey`: `["convexQuery", functionName, args]`
- `staleTime`: `Infinity` (data is pushed reactively, never "stale")
- `enabled`: `false` when args is `"skip"`

### Skip Pattern

```typescript
const { data } = useQuery(
  convexQuery(api.messages.list, channelId ? { channelId } : "skip"),
);
```

Passing `"skip"` sets `enabled: false` on the TanStack query options.

### Queries Without Arguments

```typescript
convexQuery(api.messages.listAll); // no args needed
convexQuery(api.messages.listAll, {}); // also valid
```

### How Reactivity Works

When TanStack Query activates a Convex query key, the `ConvexQueryClient` creates a `Watch` subscription on the `ConvexReactClient`. When the server pushes new data, it calls `queryClient.setQueryData()` to update TanStack Query's cache. No polling. No manual invalidation needed after mutations.

## Non-Reactive Action Queries with `convexAction`

For Convex actions (which don't subscribe to data changes):

```typescript
import { convexAction } from "@convex-dev/react-query";

const { data } = useQuery(convexAction(api.ai.summarize, { documentId }));
```

Actions are fetched once and use standard TanStack Query refetch mechanisms (no server push). Same `"skip"` support as `convexQuery`.

## Mutations

Use `useConvexMutation` (re-exported from `convex/react`) as the `mutationFn` for TanStack's `useMutation`:

```typescript
import { useMutation } from '@tanstack/react-query';
import { useConvexMutation } from '@convex-dev/react-query';

function SendButton({ channelId }: { channelId: string }) {
    const { mutate, isPending } = useMutation({
        mutationFn: useConvexMutation(api.messages.send),
    });

    return (
        <button
            disabled={isPending}
            onClick={() => mutate({ body: 'Hello', channelId })}
        >
            Send
        </button>
    );
}
```

**No `onSuccess` invalidation needed.** Because Convex queries are reactive subscriptions, the cache updates automatically when the mutation changes server data.

## Actions in Mutations

```typescript
import { useConvexAction } from "@convex-dev/react-query";

const { mutate } = useMutation({
  mutationFn: useConvexAction(api.ai.generateReport),
});
```

## Suspense

Works with `useSuspenseQuery`:

```typescript
import { useSuspenseQuery } from '@tanstack/react-query';
import { convexQuery } from '@convex-dev/react-query';

function Messages({ channelId }: { channelId: string }) {
    const { data } = useSuspenseQuery(
        convexQuery(api.messages.list, { channelId })
    );
    // data is guaranteed non-undefined
    return data.map((msg) => <Message key={msg._id} message={msg} />);
}
```

## SSR / Prefetching

The `ConvexQueryClient` has built-in SSR support. On the server, it creates a `ConvexHttpClient` internally.

### SSR Consistency Modes

```typescript
// Consistent (default) — two roundtrips, all queries see same DB snapshot
const convexQueryClient = new ConvexQueryClient(convexClient);

// Faster but inconsistent — single roundtrip, queries may see different snapshots
const convexQueryClient = new ConvexQueryClient(convexClient, {
  dangerouslyUseInconsistentQueriesDuringSSR: true,
});
```

### Prefetching

```typescript
await queryClient.prefetchQuery(
  convexQueryClient.queryOptions(api.messages.list, { channelId }),
);
```

## All Exports

```typescript
// Package exports
export { ConvexQueryClient }; // Class — bridges Convex and TanStack Query
export { convexQuery }; // Function — reactive query options factory
export { convexAction }; // Function — non-reactive action options factory

// Re-exports from convex/react (convenience)
export { useConvexQuery }; // useQuery from convex/react
export { useConvexQueries }; // useQueries from convex/react
export { useConvexPaginatedQuery }; // usePaginatedQuery from convex/react
export { useConvexMutation }; // useMutation from convex/react
export { useConvexAction }; // useAction from convex/react
export { useConvex }; // useConvex from convex/react
export { useConvexAuth }; // useConvexAuth from convex/react
export { optimisticallyUpdateValueInPaginatedQuery }; // from convex/react
```

## Limitations

- **Paginated queries** — no TanStack-specific pagination wrapper (no `convexInfiniteQuery`). Use `useConvexPaginatedQuery` (re-exported from `convex/react`) for pagination.
- **Optimistic updates** — use `useConvexMutation` with `.withOptimisticUpdate()` from `convex/react`, not TanStack's optimistic update API.
- **Auth edge cases** — JWT handling uses automatic retry on auth state changes, but edge cases may exist.
