# Convex Triggers & Real-Time Stats Patterns

How to use `convex-helpers` triggers to react to data changes, and how to combine triggers with aggregates and migrations for real-time statistics.

For component-specific API details, see the convex-components skill references:

- **Aggregates**: `convex-components/references/AGGREGATE.md`
- **Sharded Counters**: `convex-components/references/SHARDED-COUNTER.md`
- **Migrations**: `convex-components/references/MIGRATIONS.md`

---

## Sharded Counters vs Aggregates

| Need                                            | Use             | Package                       |
| ----------------------------------------------- | --------------- | ----------------------------- |
| Total count/sum (high write throughput)         | Sharded Counter | `@convex-dev/sharded-counter` |
| Ordering / ranking / pagination by computed key | Aggregate       | `@convex-dev/aggregate`       |

Rule of thumb: **Totals → sharded counters**, **Ranked lists → aggregates**

---

## Triggers

Triggers run automatically when data in a table changes. They execute **atomically within the same mutation** — queries running in parallel will never see a state where data changed but the trigger didn't run.

They intercept `insert`, `patch`, `replace`, and `delete` operations.

### Installation

```bash
bun add convex-helpers
```

### Setup

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

// IMPORTANT: Import the file where triggers are registered (must be AFTER exports)
import "./triggers";
```

**Critical:** All mutation files must import from `./functions`, not `_generated/server`:

```typescript
import { mutation } from "./functions"; // ✅ triggers fire
// import { mutation } from './_generated/server'; // ❌ triggers WON'T fire
```

### Registering Triggers

```typescript
// convex/triggers.ts
import { triggers } from "./functions";

// Register aggregate triggers (auto-sync)
triggers.register("orders", ordersByStatus.trigger());
triggers.register("orders", revenueByProduct.trigger());

// Custom trigger — the change object
triggers.register("tableName", async (ctx, change) => {
  // change.id: Document ID
  // change.operation: 'insert' | 'patch' | 'replace' | 'delete'
  // change.newDoc: New document state (present on insert, patch, replace)
  // change.oldDoc: Old document state (present on patch, replace, delete)
});
```

### Common Trigger Patterns

#### Detecting Field Changes

```typescript
triggers.register("orders", async (ctx, change) => {
  if (change.operation === "update") {
    if (change.oldDoc.status !== change.newDoc.status) {
      await ctx.scheduler.runAfter(0, internal.notifications.sendStatusUpdate, {
        orderId: change.id,
        oldStatus: change.oldDoc.status,
        newStatus: change.newDoc.status,
      });
    }
  }
});
```

#### Cascading Deletes

```typescript
triggers.register("projects", async (ctx, change) => {
  if (change.operation === "delete") {
    const tasks = await ctx.db
      .query("tasks")
      .withIndex("by_project", (q) => q.eq("projectId", change.oldDoc._id))
      .collect();
    for (const task of tasks) {
      await ctx.db.delete(task._id);
    }
  }
});
```

#### Resource Cleanup

```typescript
triggers.register("files", async (ctx, change) => {
  if (change.operation === "delete" && change.oldDoc.storageId) {
    await ctx.scheduler.runAfter(0, internal.storage.deleteFile, {
      storageId: change.oldDoc.storageId,
    });
  }
});
```

#### Audit Logging

```typescript
triggers.register("sensitiveData", async (ctx, change) => {
  await ctx.db.insert("auditLog", {
    tableName: "sensitiveData",
    operation: change.operation,
    documentId: change.id,
    oldValue: change.oldDoc ? JSON.stringify(change.oldDoc) : null,
    newValue: change.newDoc ? JSON.stringify(change.newDoc) : null,
    timestamp: Date.now(),
  });
});
```

#### Denormalizing Fields

Keep derived fields in sync automatically:

```typescript
triggers.register("books", async (ctx, change) => {
  if (change.newDoc) {
    const allFields =
      change.newDoc.title +
      " " +
      change.newDoc.author +
      " " +
      change.newDoc.summary;
    if (change.newDoc.allFields !== allFields) {
      await ctx.db.patch(change.id, { allFields });
    }
  }
});
```

#### Data Validation

Reject invalid data by throwing — the entire mutation rolls back:

```typescript
triggers.register("users", async (ctx, change) => {
  if (change.newDoc) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(change.newDoc.email)) {
      throw new Error(`invalid email ${change.newDoc.email}`);
    }
  }
});
```

#### Write Authorization

Enforce row-level security on writes:

```typescript
triggers.register("messages", async (ctx, change) => {
  const user = await getAuthedUser(ctx);
  const owner = change.oldDoc?.owner ?? change.newDoc?.owner;
  if (user !== owner) {
    throw new Error(
      `user ${user} is not allowed to modify message owned by ${owner}`,
    );
  }
});
```

#### Debounced Async Processing

Cancel previously scheduled work if the document changes again quickly:

```typescript
const scheduled: Record<string, Id<"_scheduled_functions">> = {};
triggers.register("users", async (ctx, change) => {
  if (scheduled[change.id]) {
    await ctx.scheduler.cancel(scheduled[change.id]);
  }
  scheduled[change.id] = await ctx.scheduler.runAfter(
    0,
    internal.users.syncToExternalService,
    { id: change.id, user: change.newDoc },
  );
});
```

### Caveats

**Error catching**: Triggers execute after data modification. If a trigger throws, the entire mutation rolls back. However, if the mutation **catches** the error, the data modification still commits:

```typescript
// ⚠️ The patch commits even though the trigger threw, because the error is caught
export const riskyUpdate = mutation({
  handler: async (ctx, { id, body }) => {
    try {
      await ctx.db.patch(id, { body });
    } catch (e) {
      console.error("trigger rejected the write");
      // patch is still committed!
    }
  },
});
```

**Triggers don't run for**: plain mutations from `_generated/server`, Convex dashboard direct edits, `npx convex import`, or streaming imports.

**Mutation size limits**: Triggers can cascade (trigger A deletes docs → trigger B fires). Large cascades may hit mutation size limits.

### ESLint: Enforce Wrapper Usage

```json
"no-restricted-imports": [
  "error",
  {
    "patterns": [{
      "group": ["*/_generated/server"],
      "importNames": ["mutation", "internalMutation"],
      "message": "Use functions.ts for mutation"
    }]
  }
]
```

---

## Combining Aggregates + Triggers + Migrations

The full pattern for real-time statistics:

1. **Aggregates** define what to count/sum
2. **Triggers** keep aggregates in sync on every write
3. **Migrations** backfill aggregates for existing data

### File Structure

```
convex/
├── convex.config.ts      # Register aggregate + migration components
├── functions.ts          # Create triggers, export wrapped mutation
├── aggregates.ts         # Define TableAggregate instances
├── triggers.ts           # Register aggregate.trigger() + custom triggers
├── migrations.ts         # Define backfill migrations
└── stats.ts              # Query functions using aggregates
```

### Complete Example

**1. Config:**

```typescript
// convex/convex.config.ts
import { defineApp } from "convex/server";
import aggregate from "@convex-dev/aggregate/convex.config";
import migrations from "@convex-dev/migrations/convex.config";

const app = defineApp();
app.use(aggregate, { name: "tasksByProject" });
app.use(aggregate, { name: "tasksByStatus" });
app.use(migrations);
export default app;
```

**2. Aggregates:**

```typescript
// convex/aggregates.ts
import { TableAggregate } from "@convex-dev/aggregate";
import { components } from "./_generated/api";
import type { DataModel, Id } from "./_generated/dataModel";

export const tasksByProject = new TableAggregate<{
  Key: Id<"projects">;
  DataModel: DataModel;
  TableName: "tasks";
}>(components.tasksByProject, { sortKey: (doc) => doc.projectId });

export const tasksByStatus = new TableAggregate<{
  Key: [Id<"projects">, string];
  DataModel: DataModel;
  TableName: "tasks";
}>(components.tasksByStatus, { sortKey: (doc) => [doc.projectId, doc.status] });
```

**3. Triggers:**

```typescript
// convex/triggers.ts
import { triggers } from "./functions";
import { tasksByProject, tasksByStatus } from "./aggregates";

triggers.register("tasks", tasksByProject.trigger());
triggers.register("tasks", tasksByStatus.trigger());
```

**4. Migrations (backfill existing data):**

```typescript
// convex/migrations.ts
import { Migrations } from "@convex-dev/migrations";
import { components, internal } from "./_generated/api";
import type { DataModel } from "./_generated/dataModel";
import { tasksByProject, tasksByStatus } from "./aggregates";

export const migrations = new Migrations<DataModel>(components.migrations);
export const run = migrations.runner();

export const backfillTaskAggregates = migrations.define({
  table: "tasks",
  migrateOne: async (ctx, doc) => {
    await tasksByProject.insertIfDoesNotExist(ctx, doc);
    await tasksByStatus.insertIfDoesNotExist(ctx, doc);
  },
});
export const runTaskBackfill = migrations.runner(
  internal.migrations.backfillTaskAggregates,
);
```

**5. Query stats:**

```typescript
// convex/stats.ts
import { query } from "./_generated/server";
import { v } from "convex/values";
import { tasksByProject, tasksByStatus } from "./aggregates";

export const getTaskStats = query({
  args: { projectId: v.id("projects") },
  handler: async (ctx, { projectId }) => {
    const [total, todo, done] = await Promise.all([
      tasksByProject.count(ctx, {
        bounds: {
          lower: { key: projectId, inclusive: true },
          upper: { key: projectId, inclusive: true },
        },
      }),
      tasksByStatus.count(ctx, {
        bounds: {
          lower: { key: [projectId, "todo"], inclusive: true },
          upper: { key: [projectId, "todo"], inclusive: true },
        },
      }),
      tasksByStatus.count(ctx, {
        bounds: {
          lower: { key: [projectId, "done"], inclusive: true },
          upper: { key: [projectId, "done"], inclusive: true },
        },
      }),
    ]);
    return { total, todo, done };
  },
});
```

---

## Best Practices

- **Always use wrapped mutations** — import from `./functions`, never `_generated/server`
- **Keep triggers fast** — schedule expensive work with `ctx.scheduler.runAfter(0, ...)`
- **Make migrations idempotent** — use `insertIfDoesNotExist`, not `insert`
- **Clear before re-backfilling** — call `aggregate.clear(ctx)` then re-run migration if aggregates get out of sync
- **Monitor performance** — watch for slow mutations (too many triggers) and out-of-sync aggregates after deploys
