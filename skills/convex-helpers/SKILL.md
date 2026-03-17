---
name: convex-helpers
description: Discover and use convex-helpers utilities for relationships, filtering, sessions, custom functions, CRUD generation, validator utilities, cached queries, and more. Use when you need pre-built Convex patterns, typed validators (brandedString, literals, nullable, pick, omit, typedV), auto-generated CRUD, query caching, or useQueryWithStatus.
---

# Convex Helpers Guide

Use convex-helpers to add common patterns and utilities to your Convex backend without reinventing the wheel.

## What is convex-helpers?

`convex-helpers` is the official collection of utilities that complement Convex. It provides battle-tested patterns for common backend needs.

**Installation:**

```bash
npm install convex-helpers
```

## Available Helpers

### 1. Relationship Helpers

Traverse relationships between tables in a readable, type-safe way.

**Use when:**

- Loading related data across tables
- Following foreign key relationships
- Building nested data structures

**Example:**

```typescript
import { getOneFrom, getManyFrom } from "convex-helpers/server/relationships";

export const getTaskWithUser = query({
  args: { taskId: v.id("tasks") },
  handler: async (ctx, args) => {
    const task = await ctx.db.get(args.taskId);
    if (!task) return null;

    // Get related user
    const user = await getOneFrom(ctx.db, "users", "by_id", task.userId, "_id");

    // Get related comments
    const comments = await getManyFrom(
      ctx.db,
      "comments",
      "by_task",
      task._id,
      "taskId",
    );

    return { ...task, user, comments };
  },
});
```

**Key Functions:**

- `getOneFrom` - Get single related document
- `getManyFrom` - Get multiple related documents
- `getManyVia` - Get many-to-many relationships through junction table

### 2. Custom Functions (Data Protection) ⭐ MOST IMPORTANT

**This is Convex's alternative to Row Level Security (RLS).** Instead of database-level policies, use custom function wrappers to automatically add auth and access control to all queries and mutations.

Create wrapped versions of query/mutation/action with custom behavior.

**Use when:**

- **Data protection and access control** (PRIMARY USE CASE)
- Want to add auth logic to all functions
- Multi-tenant applications
- Role-based access control (RBAC)
- Need to inject common data into ctx
- Building internal-only functions
- Adding logging/monitoring to all functions

**Why this instead of RLS:**

- ✅ TypeScript, not SQL policies
- ✅ Full type safety
- ✅ Easy to test and debug
- ✅ More flexible than database policies
- ✅ Works across your entire backend

**Example: Custom Query with Auto-Auth**

```typescript
// convex/lib/customFunctions.ts
import { customQuery } from "convex-helpers/server/customFunctions";
import { query } from "../_generated/server";

export const authenticatedQuery = customQuery(query, {
  args: {}, // No additional args required
  input: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) {
      throw new Error("Not authenticated");
    }

    const user = await ctx.db
      .query("users")
      .withIndex("by_token", (q) =>
        q.eq("tokenIdentifier", identity.tokenIdentifier),
      )
      .unique();

    if (!user) throw new Error("User not found");

    // Add user to context
    return { ctx: { ...ctx, user }, args };
  },
});

// Usage in your functions
export const getMyTasks = authenticatedQuery({
  handler: async (ctx) => {
    // ctx.user is automatically available!
    return await ctx.db
      .query("tasks")
      .withIndex("by_user", (q) => q.eq("userId", ctx.user._id))
      .collect();
  },
});
```

**Example: Multi-Tenant Data Protection**

```typescript
import { customQuery } from "convex-helpers/server/customFunctions";
import { query } from "../_generated/server";

// Organization-scoped query - automatic access control
export const orgQuery = customQuery(query, {
  args: { orgId: v.id("organizations") },
  input: async (ctx, args) => {
    const user = await getCurrentUser(ctx);

    // Verify user is a member of this organization
    const member = await ctx.db
      .query("organizationMembers")
      .withIndex("by_org_and_user", (q) =>
        q.eq("orgId", args.orgId).eq("userId", user._id),
      )
      .unique();

    if (!member) {
      throw new Error("Not authorized for this organization");
    }

    // Inject org context
    return {
      ctx: {
        ...ctx,
        user,
        orgId: args.orgId,
        role: member.role,
      },
      args,
    };
  },
});

// Usage - data automatically scoped to organization
export const getOrgProjects = orgQuery({
  args: { orgId: v.id("organizations") },
  handler: async (ctx) => {
    // ctx.user and ctx.orgId automatically available and verified!
    return await ctx.db
      .query("projects")
      .withIndex("by_org", (q) => q.eq("orgId", ctx.orgId))
      .collect();
  },
});
```

**Example: Role-Based Access Control**

```typescript
import { customMutation } from "convex-helpers/server/customFunctions";
import { mutation } from "../_generated/server";

export const adminMutation = customMutation(mutation, {
  args: {},
  input: async (ctx, args) => {
    const user = await getCurrentUser(ctx);

    if (user.role !== "admin") {
      throw new Error("Admin access required");
    }

    return { ctx: { ...ctx, user }, args };
  },
});

// Usage - only admins can call this
export const deleteUser = adminMutation({
  args: { userId: v.id("users") },
  handler: async (ctx, args) => {
    // Only admins reach this code
    await ctx.db.delete(args.userId);
  },
});
```

### 3. Filter Helper

Apply complex TypeScript filters to database queries.

**Use when:**

- Need to filter by computed values
- Filtering logic is too complex for indexes
- Working with small result sets

**Example:**

```typescript
import { filter } from "convex-helpers/server/filter";

export const getActiveTasks = query({
  handler: async (ctx) => {
    const now = Date.now();
    const threeDaysAgo = now - 3 * 24 * 60 * 60 * 1000;

    return await filter(
      ctx.db.query("tasks"),
      (task) =>
        !task.completed &&
        task.createdAt > threeDaysAgo &&
        task.priority === "high",
    ).collect();
  },
});
```

**Note:** Still prefer indexes when possible! Use filter for complex logic that can't be indexed.

### 4. Sessions

Track users across requests even when not logged in.

**Use when:**

- Need to track anonymous users
- Building shopping cart for guests
- Tracking user behavior before signup
- A/B testing without auth

**Server setup — create a session-aware query builder:**

```typescript
// convex/lib/sessions.ts
import { customQuery } from "convex-helpers/server/customFunctions";
import { SessionIdArg } from "convex-helpers/server/sessions";
import { query } from "../_generated/server";

export const queryWithSession = customQuery(query, {
  args: SessionIdArg,
  input: async (ctx, { sessionId }) => {
    const anonymousUser = await getAnonUser(ctx, sessionId);
    return { ctx: { ...ctx, anonymousUser }, args: {} };
  },
});
```

**Usage in functions:**

```typescript
export const mySessionQuery = queryWithSession({
  args: { arg1: v.number() },
  handler: async (ctx, args) => {
    // ctx.anonymousUser is available
  },
});
```

**Client (React) — wrap app with SessionProvider:**

```typescript
import { SessionProvider } from "convex-helpers/react/sessions";

// In your app root:
<ConvexProvider client={convex}>
  <SessionProvider>
    <App />
  </SessionProvider>
</ConvexProvider>
```

**Client hooks:**

```typescript
import { useSessionQuery } from "convex-helpers/react/sessions";

const results = useSessionQuery(api.myModule.mySessionQuery, { arg1: 1 });
```

### 5. Zod Validation

Use Zod schemas instead of Convex validators.

**Use when:**

- Already using Zod in your project
- Want more complex validation logic
- Need custom error messages

**Example:**

```typescript
import { zCustomQuery } from "convex-helpers/server/zod";
import { z } from "zod";
import { query } from "./_generated/server";

const argsSchema = z.object({
  email: z.string().email(),
  age: z.number().min(18).max(120),
});

export const createUser = zCustomQuery(query, {
  args: argsSchema,
  handler: async (ctx, args) => {
    // args is typed from Zod schema
    return await ctx.db.insert("users", args);
  },
});
```

### 6. Alternative: Row-Level Security Helper

**Note:** Convex recommends using **custom functions** (see #2 above) as the primary data protection pattern. This RLS helper is an alternative approach that mimics traditional RLS by wrapping the database reader/writer.

**Use when:**

- Prefer RLS-style patterns from PostgreSQL
- Need to apply same rules across many functions
- Want centralized access control rules per table

**However, custom functions are usually better because:**

- More explicit (easy to see what auth is applied)
- Better error messages
- Easier to test

**Example:**

```typescript
import {
  customCtx,
  customMutation,
  customQuery,
} from "convex-helpers/server/customFunctions";
import {
  Rules,
  RLSConfig,
  wrapDatabaseReader,
  wrapDatabaseWriter,
} from "convex-helpers/server/rowLevelSecurity";
import { DataModel } from "./_generated/dataModel";
import { mutation, query, QueryCtx } from "./_generated/server";

async function rlsRules(ctx: QueryCtx) {
  const identity = await ctx.auth.getUserIdentity();
  return {
    users: {
      read: async (_, user) => {
        if (!identity && user.age < 18) return false;
        return true;
      },
      insert: async (_, user) => {
        return true;
      },
      modify: async (_, user) => {
        if (!identity)
          throw new Error("Must be authenticated to modify a user");
        return user.tokenIdentifier === identity.tokenIdentifier;
      },
    },
  } satisfies Rules<QueryCtx, DataModel>;
}

// By default, tables with no rule have `defaultPolicy` set to "allow".
const config: RLSConfig = { defaultPolicy: "deny" };

const queryWithRLS = customQuery(
  query,
  customCtx(async (ctx) => ({
    db: wrapDatabaseReader(ctx, ctx.db, await rlsRules(ctx), config),
  })),
);

const mutationWithRLS = customMutation(
  mutation,
  customCtx(async (ctx) => ({
    db: wrapDatabaseWriter(ctx, ctx.db, await rlsRules(ctx), config),
  })),
);
```

**Rules structure:** Each table gets `read`, `insert`, and `modify` async functions that return `boolean`. The `wrapDatabaseReader`/`wrapDatabaseWriter` functions intercept all DB operations and enforce these rules transparently.

**Recommended instead: Custom functions** (see #2 above) — more explicit and type-safe.

### 7. Migrations

Run stateful online data migrations. **This is a component** (`@convex-dev/migrations`), not a convex-helpers function. See the **convex-components** skill for full details.

### 8. Triggers

Execute code automatically when data changes. Triggers run **atomically within the same mutation** — they intercept `insert`, `patch`, `replace`, and `delete` operations.

**Use when:**

- Maintaining computed/denormalized fields
- Cascading deletes
- Audit logging
- Data validation on writes
- Syncing with aggregate components

**Setup — wrap mutations to enable triggers:**

```typescript
// convex/functions.ts
import {
  mutation as rawMutation,
  internalMutation as rawInternalMutation,
} from "./_generated/server";
import { DataModel } from "./_generated/dataModel";
import { Triggers } from "convex-helpers/server/triggers";
import {
  customCtx,
  customMutation,
} from "convex-helpers/server/customFunctions";

export const triggers = new Triggers<DataModel>();

export const mutation = customMutation(rawMutation, customCtx(triggers.wrapDB));
export const internalMutation = customMutation(
  rawInternalMutation,
  customCtx(triggers.wrapDB),
);

// IMPORTANT: Import trigger registrations AFTER exports
import "./triggers";
```

**Register triggers — callback receives a `change` object:**

```typescript
// convex/triggers.ts
import { triggers } from "./functions";

triggers.register("tasks", async (ctx, change) => {
  // change.id — Document ID
  // change.operation — 'insert' | 'update' | 'delete'
  // change.newDoc — New document state (present on insert, update)
  // change.oldDoc — Old document state (present on update, delete)

  if (change.operation === "insert") {
    await ctx.db.insert("notifications", {
      userId: change.newDoc.userId,
      type: "task_created",
      taskId: change.id,
    });
  }
});

// Auto-sync aggregates
triggers.register("tasks", taskAggregate.trigger());
```

**Critical:** All mutation files must import from `./functions`, NOT `_generated/server`, or triggers won't fire.

### 9. Aggregations

Efficient O(log n) count, sum, min, max operations. **This is a component** (`@convex-dev/aggregate`), not a convex-helpers function. See the **convex-components** skill for full details.

Keep aggregates in sync automatically using triggers (see #8 above):

```typescript
triggers.register("tasks", tasksByStatus.trigger());
```

### 10. CRUD Generator

Auto-generate create, read, update, and delete functions for any table. Useful for internal tools, admin panels, and prototyping — but not recommended for production without access control wrappers around it.

**Use when:**

- Bootstrapping CRUD for a table quickly
- Building admin/internal APIs
- Prototyping before adding custom access control

**Example:**

```typescript
// convex/users.ts
import { crud } from "convex-helpers/server/crud";
import schema from "./schema.js";

export const { create, read, update, destroy } = crud(schema, "users");

// In an action:
const user = await ctx.runQuery(internal.users.read, { id: userId });

await ctx.runMutation(internal.users.update, {
  id: userId,
  patch: { status: "inactive" },
});
```

The generated functions are typed from your schema — `create` accepts the table's fields, `update` takes `id` + `patch`, `read` and `destroy` take `id`. For production, wrap these with custom functions (see #2) to add auth.

### 11. Validator Utilities

A rich set of utilities for building, composing, and validating Convex validators beyond the basic `v.*` primitives. These help you write stricter schemas, reuse validator fragments, and validate data at runtime.

**Use when:**

- Defining schemas with branded types, enums, or nullable fields
- Picking/omitting fields from a table's validator for args or return types
- Validating data at runtime (e.g., in actions before inserting)
- Deprecating old fields without breaking the schema

**Schema-level utilities** — import from `convex-helpers/validators`:

```typescript
// convex/schema.ts
import {
  literals,
  deprecated,
  brandedString,
  nullable,
} from "convex-helpers/validators";
import { defineSchema, defineTable } from "convex/server";
import { v, Infer } from "convex/values";

export const emailValidator = brandedString("email");
export type Email = Infer<typeof emailValidator>;

export default defineSchema({
  accounts: defineTable({
    balance: nullable(v.bigint()), // v.union(v.bigint(), v.null())
    status: literals("active", "inactive"), // v.union(v.literal("active"), v.literal("inactive"))
    email: emailValidator, // typed string brand
    oldField: deprecated, // accepts any value, signals "stop using this"
  }).index("status", ["status"]),
});
```

**Function-level utilities** — `typedV`, `doc`, `partial`, `pick`, `omit`, `validate`:

```typescript
import { doc, typedV, partial } from "convex-helpers/validators";
import { omit, pick } from "convex-helpers";
import schema from "./schema";

// Schema-aware validator builder — gives you vv.id("accounts"), vv.doc("accounts"), etc.
const vv = typedV(schema);

export const replaceUser = internalMutation({
  args: {
    id: vv.id("accounts"),
    replace: vv.object({
      ...schema.tables.accounts.validator.fields,
      ...partial(systemFields("accounts")),
    }),
  },
  returns: doc(schema, "accounts"),
  handler: async (ctx, args) => {
    await ctx.db.replace(args.id, args.replace);
    return await ctx.db.get(args.id);
  },
});

// Pick specific fields from a table's validator
const balanceAndEmail = pick(vv.doc("accounts").fields, ["balance", "email"]);

// Omit fields
const accountWithoutBalance = omit(vv.doc("accounts").fields, ["balance"]);

// Runtime validation — useful in actions before inserting
validate(balanceAndEmail, value); // returns boolean
validate(balanceAndEmail, value, { throw: true }); // throws ValidationError
validate(vv.id("accounts"), id, { db: ctx.db }); // validates ID exists in table
```

**Key functions:**

- `brandedString(name)` — typed string brand (e.g., emails, slugs)
- `literals(...values)` — union of string/number literals (enum-like)
- `nullable(validator)` — shorthand for `v.union(validator, v.null())`
- `deprecated` — marks a field as deprecated in schema
- `partial(fields)` — makes all fields optional
- `typedV(schema)` — schema-aware validator builder with `.id()`, `.doc()`, `.object()`
- `doc(schema, table)` — full document validator including system fields
- `pick(fields, keys)` / `omit(fields, keys)` — subset/exclude fields
- `validate(validator, value, opts?)` — runtime validation with optional throw

### 12. Cached Queries (React)

Drop-in replacement for Convex's `useQuery` that caches results client-side. When a component unmounts and remounts, it gets the cached value instantly instead of showing a loading state. Useful for tabs, navigation, and any UI where users revisit the same data.

**Use when:**

- Users navigate between views that re-fetch the same data
- You want instant UI on remount instead of loading spinners
- Tab-based UIs, sidebars, or detail panels

**Setup — wrap your app with the cache provider:**

```typescript
import { ConvexQueryCacheProvider } from "convex-helpers/react/cache";
// For Next.js: import from "convex-helpers/react/cache/provider";

export default function App({ children }) {
  return (
    <ConvexClientProvider>
      <ConvexQueryCacheProvider>
        {children}
      </ConvexQueryCacheProvider>
    </ConvexClientProvider>
  );
}
```

**Usage — swap your import:**

```typescript
// Before:
// import { useQuery } from "convex/react";

// After:
import { useQuery } from "convex-helpers/react/cache";
// For Next.js: import from "convex-helpers/react/cache/hooks";

const users = useQuery(api.users.list);
```

Also provides cached versions of `useQueries` and `usePaginatedQuery` from the same module. Default cache expiration is 5 minutes, max 250 idle entries.

### 13. useQueryWithStatus (React)

Richer alternative to the standard `useQuery` that returns explicit status fields instead of `undefined` for loading and throwing on errors. Makes it easier to build UIs with proper loading/error states without try-catch.

**Use when:**

- You want to distinguish "loading" from "no data" (standard `useQuery` returns `undefined` for both)
- You need error objects instead of thrown exceptions
- Building UIs with loading/error/success states

**Setup — create the hook once:**

```typescript
// lib/hooks.ts
import { makeUseQueryWithStatus } from "convex-helpers/react";
import { useQueries } from "convex/react";

export const useQueryWithStatus = makeUseQueryWithStatus(useQueries);
```

**Usage:**

```typescript
import { useQueryWithStatus } from "@/lib/hooks";

function TaskList() {
  const { status, data, error, isSuccess, isPending, isError } =
    useQueryWithStatus(api.tasks.list, { userId: "123" });

  if (isPending) return <Spinner />;
  if (isError) return <ErrorMessage error={error} />;
  return <TaskTable tasks={data} />;
}
```

Returns `{ status, data, error, isSuccess, isPending, isError }` — similar to TanStack Query's API shape, so it feels familiar if you've used that.

## Common Patterns

### Pattern 1: Authenticated Queries with User Context

```typescript
import { customQuery } from "convex-helpers/server/customFunctions";

export const authedQuery = customQuery(query, {
  args: {},
  input: async (ctx, args) => {
    const user = await getCurrentUser(ctx);
    return { ctx: { ...ctx, user }, args };
  },
});

// Now all queries automatically have user in context
export const getMyData = authedQuery({
  handler: async (ctx) => {
    // ctx.user is typed and available!
    return await ctx.db
      .query("data")
      .withIndex("by_user", (q) => q.eq("userId", ctx.user._id))
      .collect();
  },
});
```

### Pattern 2: Loading Related Data

```typescript
import { getOneFrom, getManyFrom } from "convex-helpers/server/relationships";

export const getPostWithDetails = query({
  args: { postId: v.id("posts") },
  handler: async (ctx, args) => {
    const post = await ctx.db.get(args.postId);
    if (!post) return null;

    // Load author
    const author = await getOneFrom(
      ctx.db,
      "users",
      "by_id",
      post.authorId,
      "_id",
    );

    // Load comments
    const comments = await getManyFrom(
      ctx.db,
      "comments",
      "by_post",
      post._id,
      "postId",
    );

    // Load tags (many-to-many)
    const tagLinks = await getManyFrom(
      ctx.db,
      "postTags",
      "by_post",
      post._id,
      "postId",
    );

    const tags = await Promise.all(
      tagLinks.map((link) =>
        getOneFrom(ctx.db, "tags", "by_id", link.tagId, "_id"),
      ),
    );

    return { ...post, author, comments, tags };
  },
});
```

### Pattern 3: Batch Operations with Error Handling

```typescript
import { asyncMap } from "convex-helpers";

export const batchUpdateTasks = mutation({
  args: {
    taskIds: v.array(v.id("tasks")),
    status: v.string(),
  },
  handler: async (ctx, args) => {
    const results = await asyncMap(args.taskIds, async (taskId) => {
      try {
        const task = await ctx.db.get(taskId);
        if (task) {
          await ctx.db.patch(taskId, { status: args.status });
          return { success: true, taskId };
        }
        return { success: false, taskId, error: "Not found" };
      } catch (error) {
        return { success: false, taskId, error: error.message };
      }
    });

    return results;
  },
});
```

## Best Practices

1. **Start with convex-helpers**
   - Don't reinvent common patterns
   - Use battle-tested utilities
   - Contribute back if you build something useful

2. **Custom Functions for Auth**
   - Create `authedQuery`, `authedMutation`, etc.
   - Inject user context automatically
   - Reduces boilerplate

3. **Relationships Over Nesting**
   - Use relationship helpers
   - Keep data normalized
   - Load related data as needed

4. **Filter Sparingly**
   - Prefer indexes when possible
   - Use filter for complex computed logic
   - Good for small result sets

5. **Sessions for Anonymous Users**
   - Track before signup
   - Migrate to user account later
   - Great for cart, preferences, etc.

## Documentation

- [convex-helpers GitHub](https://github.com/get-convex/convex-helpers)
- [convex-helpers on npm](https://www.npmjs.com/package/convex-helpers)
- [Relationship Helpers Guide](https://stack.convex.dev/functional-relationships-helpers)

## When to Use What

| Need                    | Use                                     | Import From                                                          |
| ----------------------- | --------------------------------------- | -------------------------------------------------------------------- |
| Load related data       | `getOneFrom`, `getManyFrom`             | `convex-helpers/server/relationships`                                |
| Auth in all functions   | `customQuery`                           | `convex-helpers/server/customFunctions`                              |
| Complex filters         | `filter`                                | `convex-helpers/server/filter`                                       |
| Anonymous users         | `useSessionId`                          | `convex-helpers/react/sessions`                                      |
| Zod validation          | `zCustomQuery`                          | `convex-helpers/server/zod`                                          |
| Data migrations         | `Migrations`                            | `@convex-dev/migrations` (component)                                 |
| Triggers                | `Triggers`                              | `convex-helpers/server/triggers` (requires `customMutation` wrapper) |
| Auto CRUD               | `crud`                                  | `convex-helpers/server/crud`                                         |
| Typed validators        | `brandedString`, `literals`, `nullable` | `convex-helpers/validators`                                          |
| Schema-aware validators | `typedV`, `doc`, `partial`              | `convex-helpers/validators`                                          |
| Pick/omit fields        | `pick`, `omit`                          | `convex-helpers`                                                     |
| Runtime validation      | `validate`                              | `convex-helpers/validators`                                          |
| Cached queries          | `useQuery` (cached)                     | `convex-helpers/react/cache`                                         |
| Query with status       | `makeUseQueryWithStatus`                | `convex-helpers/react`                                               |
| Batch async ops         | `asyncMap`                              | `convex-helpers`                                                     |
