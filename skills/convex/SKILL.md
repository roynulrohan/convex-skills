---
name: convex
description: "Expert guidance for Convex development: functions (queries, mutations, actions, HTTP actions), schema design, database operations, indexes, authentication, scheduling, cron jobs, file storage, full-text and vector search, error handling, runtimes, OCC avoidance, migrations, ESLint plugin, TypeScript best practices, Next.js App Router integration, React client patterns (useQuery, useMutation, usePaginatedQuery, optimistic updates, SSR), and TanStack Query integration. Use when working with convex/ directory code, ctx.db, useQuery, useMutation, useAction, usePaginatedQuery, ConvexError, internal functions, ctx.scheduler, ctx.storage, httpAction, searchIndex, vectorIndex, schema evolution, ConvexProvider, useConvexAuth, preloadQuery, @convex-dev/react-query, or any Convex-related patterns."
---

# Convex Backend Development

## Core Architecture

Convex is a reactive database where queries are TypeScript functions. The sync engine (queries + mutations + database) is the heart of Convex — center your app around it.

### The Zen of Convex

1. **Convex manages the hard parts** — Let Convex handle caching, real-time sync, and consistency
2. **Functions are the API** — Design your functions as your application's interface
3. **Schema is truth** — Define your data model explicitly in schema.ts
4. **TypeScript everywhere** — Leverage end-to-end type safety
5. **Queries are reactive** — Think in terms of subscriptions, not requests

### Function Types

| Type         | DB Access     | Deterministic | Cached/Reactive | Use For                    |
| ------------ | ------------- | ------------- | --------------- | -------------------------- |
| `query`      | Read only     | Yes           | Yes             | All reads, subscriptions   |
| `mutation`   | Read/Write    | Yes           | No              | All writes (transactions)  |
| `action`     | Via ctx.run\* | No            | No              | External APIs, LLMs, email |
| `httpAction` | Via ctx.run\* | No            | No              | Webhooks, custom HTTP      |

**Key rule**: Queries and mutations cannot make network requests. Actions cannot directly access the database.

## Code Quality — ESLint Plugin

All patterns in this skill comply with `@convex-dev/eslint-plugin`. Install it for build-time validation:

```bash
bun add -d @convex-dev/eslint-plugin
```

```js
// eslint.config.js
import { defineConfig } from "eslint/config";
import convexPlugin from "@convex-dev/eslint-plugin";

export default defineConfig([...convexPlugin.configs.recommended]);
```

| Rule                                | What it enforces                  |
| ----------------------------------- | --------------------------------- |
| `no-old-registered-function-syntax` | Object syntax with `handler`      |
| `require-argument-validators`       | `args: {}` on all functions       |
| `explicit-table-ids`                | Table name in db operations       |
| `import-wrong-runtime`              | No Node imports in Convex runtime |

Docs: https://docs.convex.dev/eslint

## Project Structure (Best Practice)

```
convex/
├── _generated/         # Auto-generated types (commit this)
├── schema.ts           # Database schema
├── model/              # Helper functions (most logic lives here)
│   ├── users.ts
│   └── messages.ts
├── users.ts            # Thin wrappers exposing public API
├── messages.ts
├── crons.ts            # Cron job definitions
└── http.ts             # HTTP action routes
```

## Essential Patterns

### 1. Function Structure

```typescript
// convex/messages.ts
import { query, mutation, internalMutation } from "./_generated/server";
import { internal } from "./_generated/api";
import { v } from "convex/values";

// PUBLIC query with validators (always validate public functions)
export const list = query({
  args: { channelId: v.id("channels") },
  handler: async (ctx, { channelId }) => {
    return await ctx.db
      .query("messages")
      .withIndex("by_channel", (q) => q.eq("channelId", channelId))
      .order("desc")
      .take(50);
  },
});

// PUBLIC mutation with validators and auth check
export const send = mutation({
  args: { channelId: v.id("channels"), body: v.string() },
  handler: async (ctx, { channelId, body }) => {
    const user = await ctx.auth.getUserIdentity();
    if (!user) throw new Error("Unauthorized");

    await ctx.db.insert("messages", {
      channelId,
      body,
      authorId: user.subject,
    });
  },
});

// INTERNAL mutation (for scheduling, crons, actions)
export const deleteOld = internalMutation({
  args: { before: v.number() },
  handler: async (ctx, { before }) => {
    const old = await ctx.db
      .query("messages")
      .withIndex("by_createdAt", (q) => q.lt("_creationTime", before))
      .take(100);
    for (const msg of old) {
      await ctx.db.delete(msg._id);
    }
  },
});
```

### 2. Internal vs Public Functions

```typescript
// Public function — exposed to clients, requires auth
export const getUser = query({
  args: { userId: v.id("users") },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");
    return await ctx.db.get(args.userId);
  },
});

// Internal function — only callable from other Convex functions
export const updateUserStats = internalMutation({
  args: { userId: v.id("users") },
  handler: async (ctx, args) => {
    // No auth check needed — internal functions are trusted
    // ...
  },
});
```

### 3. Helper Functions Pattern

Most logic should live in helper functions, NOT in query/mutation handlers:

```typescript
// convex/model/users.ts
import { QueryCtx, MutationCtx } from "../_generated/server";
import { Doc } from "../_generated/dataModel";

export async function getCurrentUser(
  ctx: QueryCtx,
): Promise<Doc<"users"> | null> {
  const identity = await ctx.auth.getUserIdentity();
  if (!identity) return null;

  return await ctx.db
    .query("users")
    .withIndex("by_tokenIdentifier", (q) =>
      q.eq("tokenIdentifier", identity.tokenIdentifier),
    )
    .unique();
}

export async function requireUser(ctx: QueryCtx): Promise<Doc<"users">> {
  const user = await getCurrentUser(ctx);
  if (!user) throw new Error("Unauthorized");
  return user;
}
```

### 4. Actions with Scheduling

```typescript
// convex/ai.ts
import { action, internalMutation } from './_generated/server';
import { internal } from './_generated/api';
import { v } from 'convex/values';

export const summarize = action({
  args: { documentId: v.id('documents') },
  handler: async (ctx, { documentId }) => {
    // Read data via internal query
    const doc = await ctx.runQuery(internal.documents.get, { documentId });

    // Call external API
    const response = await fetch('https://api.openai.com/v1/...', {...});
    const summary = await response.json();

    // Write result via internal mutation
    await ctx.runMutation(internal.documents.setSummary, {
      documentId,
      summary: summary.text
    });
  }
});

// Trigger action from mutation (not directly from client)
export const requestSummary = mutation({
  args: { documentId: v.id('documents') },
  handler: async (ctx, { documentId }) => {
    const user = await ctx.auth.getUserIdentity();
    if (!user) throw new Error('Unauthorized');

    await ctx.db.patch(documentId, { status: 'processing' });

    // Schedule action (runs after mutation commits)
    await ctx.scheduler.runAfter(0, internal.ai.summarizeInternal, {
      documentId
    });
  }
});
```

### 5. Application Errors

```typescript
import { ConvexError } from "convex/values";

export const assignRole = mutation({
  args: { roleId: v.id("roles"), userId: v.id("users") },
  handler: async (ctx, { roleId, userId }) => {
    const existing = await ctx.db
      .query("assignments")
      .withIndex("by_role", (q) => q.eq("roleId", roleId))
      .first();

    if (existing) {
      throw new ConvexError({
        code: "ROLE_TAKEN",
        message: "Role is already assigned",
      });
    }

    await ctx.db.insert("assignments", { roleId, userId });
  },
});
```

### 6. Avoiding Write Conflicts (OCC)

Convex uses Optimistic Concurrency Control. Minimize conflicts with these patterns:

```typescript
// GOOD: Make mutations idempotent
export const completeTask = mutation({
  args: { taskId: v.id("tasks") },
  handler: async (ctx, args) => {
    const task = await ctx.db.get(args.taskId);

    // Early return if already complete (idempotent)
    if (!task || task.status === "completed") return;

    await ctx.db.patch(args.taskId, {
      status: "completed",
      completedAt: Date.now(),
    });
  },
});

// GOOD: Patch directly without reading first when possible
export const updateNote = mutation({
  args: { id: v.id("notes"), content: v.string() },
  handler: async (ctx, args) => {
    // ctx.db.patch throws if document doesn't exist
    await ctx.db.patch(args.id, { content: args.content });
  },
});

// GOOD: Use Promise.all for parallel independent updates
export const reorderItems = mutation({
  args: { itemIds: v.array(v.id("items")) },
  handler: async (ctx, args) => {
    await Promise.all(
      args.itemIds.map((id, index) => ctx.db.patch(id, { order: index })),
    );
  },
});
```

### 7. Complete CRUD Pattern

```typescript
// convex/tasks.ts
import { query, mutation } from "./_generated/server";
import { v } from "convex/values";

export const list = query({
  args: { userId: v.id("users") },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("tasks")
      .withIndex("by_user", (q) => q.eq("userId", args.userId))
      .collect();
  },
});

export const create = mutation({
  args: { title: v.string(), userId: v.id("users") },
  returns: v.id("tasks"),
  handler: async (ctx, args) => {
    return await ctx.db.insert("tasks", {
      title: args.title,
      completed: false,
      userId: args.userId,
    });
  },
});

export const update = mutation({
  args: {
    taskId: v.id("tasks"),
    title: v.optional(v.string()),
    completed: v.optional(v.boolean()),
  },
  handler: async (ctx, args) => {
    const { taskId, ...updates } = args;
    const cleanUpdates = Object.fromEntries(
      Object.entries(updates).filter(([_, v]) => v !== undefined),
    );
    if (Object.keys(cleanUpdates).length > 0) {
      await ctx.db.patch(taskId, cleanUpdates);
    }
  },
});

export const remove = mutation({
  args: { taskId: v.id("tasks") },
  handler: async (ctx, args) => {
    await ctx.db.delete(args.taskId);
  },
});
```

### 8. TypeScript Best Practices

```typescript
import { Id, Doc } from "./_generated/dataModel";

// Use Id type for document references
type UserId = Id<"users">;

// Use Doc type for full documents
type User = Doc<"users">;

// Define Record types properly
const userScores: Record<Id<"users">, number> = {};
```

## Critical Rules

### DO

- Use `internal.` functions for all `ctx.run*`, `ctx.scheduler`, and crons
- Always validate args for public functions with `v.*` validators
- Always check auth in public functions: `ctx.auth.getUserIdentity()`
- Always use `.withIndex()` instead of `.filter()` for querying data
- Await all promises (enable `no-floating-promises` ESLint rule)
- Keep actions small — put logic in queries/mutations
- Batch database operations in single mutations
- Use `ConvexError` for user-facing errors
- Make mutations idempotent to handle retries gracefully
- Define return validators for functions
- Leverage TypeScript's `Id` and `Doc` types

### DON'T

- Don't use `api.` functions for scheduling (use `internal.`)
- Don't use `.filter()` — it scans all documents in memory, breaks pagination, and wastes bandwidth. Always use `.withIndex()` instead. The only exception is a known-small result set where adding an index isn't worth it
- Don't use `.collect()` on unbounded queries (use `.take()` or pagination)
- Don't use `Date.now()` in queries — while it's deterministic (frozen per invocation), reactive subscriptions won't auto-refresh as time passes, leading to stale results. Pass timestamps as arguments from the client or use `_creationTime` instead
- Don't call actions directly from client (trigger via mutation + scheduler)
- Don't make sequential `ctx.runQuery/runMutation` calls in actions (batch them)
- Don't use `ctx.runAction` unless switching runtimes (use helper functions)
- Don't read before patching unnecessarily — patch directly when possible

## Reference Guides

For detailed patterns, see:

- [FUNCTIONS.md](references/FUNCTIONS.md) — Queries, mutations, actions, internal functions
- [VALIDATION.md](references/VALIDATION.md) — Argument validation, extended validators
- [ERROR_HANDLING.md](references/ERROR_HANDLING.md) — ConvexError, application errors
- [HTTP_ACTIONS.md](references/HTTP_ACTIONS.md) — HTTP actions, CORS, webhooks
- [RUNTIMES.md](references/RUNTIMES.md) — Default vs Node.js runtime, bundling, debugging
- [DATABASE.md](references/DATABASE.md) — Schema, indexes, reading/writing data
- [SEARCH.md](references/SEARCH.md) — Full-text search, vector search, RAG patterns
- [ADVANCED.md](references/ADVANCED.md) — System tables, schema philosophy, OCC
- [AUTH.md](references/AUTH.md) — Row-level security, Convex Auth (first-party)
- [SCHEDULING.md](references/SCHEDULING.md) — Crons, scheduled functions, workflows
- [FILE_STORAGE.md](references/FILE_STORAGE.md) — Upload, store, serve, delete files
- [NEXTJS.md](references/NEXTJS.md) — Next.js App Router, SSR, Server Actions
- [MIGRATIONS.md](references/MIGRATIONS.md) — Schema migrations, backfills, field evolution
- [TRIGGERS_AND_STATS.md](references/TRIGGERS_AND_STATS.md) — convex-helpers triggers, aggregates+triggers+migrations combined patterns
- [CLIENT.md](references/CLIENT.md) — React hooks, optimistic updates, pagination, auth gates, SSR
- [TANSTACK-QUERY.md](references/TANSTACK-QUERY.md) — TanStack React Query integration with Convex

**Auth Provider Skills** (add to project as needed):

- `convex-auth` — Universal auth patterns, storing users, debugging
- `convex-clerk` — Clerk setup, webhooks, JWT configuration
- `convex-workos` — WorkOS AuthKit setup, auto-provisioning
