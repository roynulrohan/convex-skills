# React Client Patterns

Convex React hooks for queries, mutations, actions, auth, pagination, optimistic updates, and SSR.

> For TanStack Query integration, see `convex/references/TANSTACK-QUERY.md`.
> For cached queries and `useQueryWithStatus`, see the `convex-helpers` skill.

## Provider Setup

```typescript
import { ConvexProvider, ConvexReactClient } from 'convex/react';

const convex = new ConvexReactClient(import.meta.env.VITE_CONVEX_URL);

reactDOMRoot.render(
    <ConvexProvider client={convex}>
        <App />
    </ConvexProvider>
);
```

For auth providers (Clerk, Auth0, etc.), use their specialized providers (e.g. `ConvexProviderWithClerk`).

## useQuery

Subscribe to a reactive query. Automatically re-renders when underlying data changes.

```typescript
import { useQuery } from "convex/react";
import { api } from "../convex/_generated/api";

const messages = useQuery(api.messages.list, { channelId });
```

**Signature:** `useQuery(queryReference, args | "skip")`

**Returns:** `undefined` while loading, then the query's return value.

### Skip Pattern (Conditional Queries)

React hooks can't be conditional. Pass `"skip"` to disable the query without breaking hook rules:

```typescript
// Skip until we have a param
const data = useQuery(api.messages.list, channelId ? { channelId } : "skip");

// Skip until authenticated
const user = useQuery(api.users.getMe, isAuthenticated ? {} : "skip");
```

When `"skip"` is passed, the hook doesn't contact the backend and returns `undefined`.

## useMutation

```typescript
import { useMutation } from "convex/react";

const sendMessage = useMutation(api.messages.send);

// Call it (returns a promise)
await sendMessage({ body: "Hello", channelId });
```

**Signature:** `useMutation(mutationReference)` returns a stable async function.

- Automatically retries until confirmed written (idempotent — multiple retries execute only once)
- Use `try/catch` for error handling

### Optimistic Updates

Update the UI immediately while the mutation is in flight:

```typescript
const toggleTodo = useMutation(api.todos.toggle).withOptimisticUpdate(
  (localStore, args) => {
    const currentTodos = localStore.getQuery(api.todos.list, {});
    if (currentTodos !== undefined) {
      localStore.setQuery(
        api.todos.list,
        {},
        currentTodos.map((todo) =>
          todo._id === args.todoId
            ? { ...todo, completed: !todo.completed }
            : todo,
        ),
      );
    }
  },
);
```

**`localStore` API:**

- `localStore.getQuery(queryRef, args)` — current cached result, or `undefined`
- `localStore.setQuery(queryRef, args, newValue)` — set cached result locally

**Rules:**

- Never mutate objects in place — always create new objects
- Optimistic updates roll back automatically when the server result arrives

## useAction

```typescript
import { useAction } from "convex/react";

const generateSummary = useAction(api.ai.summarize);
const result = await generateSummary({ documentId });
```

**Signature:** `useAction(actionReference)` returns an async function.

- No automatic retries
- No optimistic updates
- Use for external API calls, side effects

## usePaginatedQuery

```typescript
import { usePaginatedQuery } from "convex/react";

const { results, status, loadMore, isLoading } = usePaginatedQuery(
  api.messages.list,
  { channelId }, // args (excluding paginationOpts)
  { initialNumItems: 25 }, // options
);
```

**Signature:** `usePaginatedQuery(queryRef, args | "skip", { initialNumItems })`

**Return type:**

| Property    | Type                         | Description               |
| ----------- | ---------------------------- | ------------------------- |
| `results`   | `Item[]`                     | Currently loaded results  |
| `isLoading` | `boolean`                    | Whether currently loading |
| `status`    | `PaginationStatus`           | Current pagination state  |
| `loadMore`  | `(numItems: number) => void` | Fetch more results        |

**`status` values:**

- `"LoadingFirstPage"` — loading initial data
- `"CanLoadMore"` — more items exist, call `loadMore(n)`
- `"LoadingMore"` — fetching another page
- `"Exhausted"` — reached the end

```tsx
function MessageList({ channelId }: { channelId: string }) {
  const { results, status, loadMore } = usePaginatedQuery(
    api.messages.list,
    { channelId },
    { initialNumItems: 25 },
  );

  return (
    <div>
      {results.map((msg) => (
        <Message key={msg._id} message={msg} />
      ))}
      {status === "CanLoadMore" && (
        <button onClick={() => loadMore(25)}>Load More</button>
      )}
      {status === "LoadingMore" && <Spinner />}
    </div>
  );
}
```

The query function must accept `paginationOpts`:

```typescript
// convex/messages.ts
export const list = query({
  args: {
    channelId: v.id("channels"),
    paginationOpts: paginationOptsValidator,
  },
  handler: async (ctx, { channelId, paginationOpts }) => {
    return await ctx.db
      .query("messages")
      .withIndex("by_channel", (q) => q.eq("channelId", channelId))
      .order("desc")
      .paginate(paginationOpts);
  },
});
```

## useConvexAuth

```typescript
import { useConvexAuth } from "convex/react";

const { isAuthenticated, isLoading } = useConvexAuth();
```

**Returns:** `{ isAuthenticated: boolean, isLoading: boolean }`

Use this instead of your auth provider's hook (e.g. Clerk's `useAuth()`) to ensure the token has been validated with the Convex backend.

### Auth Gate Components

```tsx
import { Authenticated, Unauthenticated, AuthLoading } from "convex/react";

function App() {
  return (
    <>
      <AuthLoading>
        <Spinner />
      </AuthLoading>
      <Unauthenticated>
        <LoginPage />
      </Unauthenticated>
      <Authenticated>
        <Dashboard />
      </Authenticated>
    </>
  );
}
```

## useConvex

Access the raw `ConvexReactClient` for one-off (non-reactive) operations:

```typescript
import { useConvex } from "convex/react";

const convex = useConvex();
const result = await convex.query(api.functions.myQuery, { id });
```

## SSR / Preloading (Next.js)

### Server-Side

```typescript
// app/page.tsx (Server Component)
import { preloadQuery } from 'convex/nextjs';
import { api } from '@/convex/_generated/api';

export default async function Page() {
    const preloaded = await preloadQuery(api.messages.list, { channelId: 'general' });
    return <MessageList preloaded={preloaded} />;
}
```

### Client-Side

```typescript
'use client';
import { usePreloadedQuery, Preloaded } from 'convex/react';
import { api } from '@/convex/_generated/api';

export function MessageList({
    preloaded,
}: {
    preloaded: Preloaded<typeof api.messages.list>;
}) {
    const messages = usePreloadedQuery(preloaded);
    // Subscribes to real-time updates after hydration
    return messages.map((msg) => <div key={msg._id}>{msg.body}</div>);
}
```

### Server-Side Utilities (Next.js)

From `"convex/nextjs"`:

```typescript
import { fetchQuery, fetchMutation, fetchAction } from "convex/nextjs";

// One-off server-side calls (Server Actions, Route Handlers)
const data = await fetchQuery(api.messages.list, { channelId });
await fetchMutation(api.messages.send, { body: "Hello", channelId });
const result = await fetchAction(api.ai.summarize, { documentId });
```

## Query Approaches Comparison

Convex offers three approaches for querying data in React. All are valid — choose based on your stack and needs:

|                          | Core `useQuery`         | `convex-helpers` cache + status            | TanStack Query                                      |
| ------------------------ | ----------------------- | ------------------------------------------ | --------------------------------------------------- |
| **Setup**                | Zero — built-in         | Provider wrapper                           | Provider + adapter                                  |
| **Real-time reactivity** | Built-in                | Built-in                                   | Built-in (via adapter)                              |
| **Loading vs no-data**   | Both return `undefined` | Explicit `isPending`/`isError`/`isSuccess` | Explicit status fields                              |
| **Caching on unmount**   | No — re-subscribes      | Yes — `ConvexQueryCacheProvider`           | Yes — `gcTime` control                              |
| **SSR/prefetching**      | `preloadQuery`          | Same as core                               | Built-in SSR support                                |
| **Devtools**             | No                      | No                                         | React Query Devtools                                |
| **Extra dependency**     | None                    | `convex-helpers`                           | `@tanstack/react-query` + `@convex-dev/react-query` |

**When to use each:**

- **Core `useQuery`** — Simple apps, Convex-only stack, no need for loading state distinction
- **`convex-helpers` cache + status** — Want `isPending`/`isError` without adding TanStack. Want cached results on remount (tabs, navigation)
- **TanStack Query** — Already using TanStack in your app, need devtools, complex cache control, or SSR patterns beyond `preloadQuery`

See the `convex-helpers` skill for `useQueryWithStatus` and `ConvexQueryCacheProvider` details.
See `convex/references/TANSTACK-QUERY.md` for the TanStack Query integration.
