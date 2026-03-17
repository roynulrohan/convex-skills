# Convex Scheduling Reference

Convex lets you schedule functions to run once or repeatedly in the future, enabling durable workflows like sending welcome emails, reconciling accounts, or cleaning up data.

**For high-scale or advanced workflows**, consider these components:

- **Workpool** — Priority queues for critical async tasks
- **Workflow** — Durable multi-step flows with retries and delays
- **Crons component** — Runtime-configurable cron schedules

## Scheduled Functions

Schedule functions to run in the future from mutations or actions. Scheduled functions are stored durably in the database and resilient to downtime.

### Basic Scheduling

```typescript
import { mutation, internalMutation } from "./_generated/server";
import { internal } from "./_generated/api";
import { v } from "convex/values";

// Schedule from a mutation
export const sendMessage = mutation({
  args: { body: v.string(), selfDestruct: v.boolean() },
  handler: async (ctx, { body, selfDestruct }) => {
    const messageId = await ctx.db.insert("messages", { body });

    if (selfDestruct) {
      // Delete after 5 seconds
      await ctx.scheduler.runAfter(
        5000, // milliseconds
        internal.messages.deleteMessage,
        { messageId },
      );
    }

    return messageId;
  },
});

// Internal function to be scheduled
export const deleteMessage = internalMutation({
  args: { messageId: v.id("messages") },
  handler: async (ctx, { messageId }) => {
    await ctx.db.delete(messageId);
  },
});
```

### Schedule Methods

```typescript
// Run after delay (milliseconds)
await ctx.scheduler.runAfter(5000, internal.jobs.process, { id: "123" });

// Run at specific time (Unix timestamp in ms, or Date)
await ctx.scheduler.runAt(
  Date.now() + 86400000, // Tomorrow
  internal.jobs.remind,
  { userId },
);

await ctx.scheduler.runAt(
  new Date("2024-12-25T00:00:00Z"),
  internal.jobs.sendHolidayGreeting,
  {},
);
```

### Scheduling from Mutations (Atomic)

Scheduling from mutations is **atomic** with the rest of the mutation:

- If the mutation succeeds → function is guaranteed to be scheduled
- If the mutation fails → no function is scheduled (even if scheduling call already executed)

```typescript
export const createOrder = mutation({
  args: { items: v.array(v.id("products")) },
  handler: async (ctx, { items }) => {
    const orderId = await ctx.db.insert("orders", { items, status: "pending" });

    // This scheduling is atomic with the insert above
    // If insert fails, this won't be scheduled
    await ctx.scheduler.runAfter(
      30 * 60 * 1000, // 30 minutes
      internal.orders.checkPayment,
      { orderId },
    );

    return orderId;
  },
});
```

### Scheduling from Actions (Non-Atomic)

Actions don't execute as a single transaction. Scheduling from actions does **not** depend on the action's outcome:

- Scheduled functions run even if the action later fails
- Use this intentionally for "fire and forget" patterns

```typescript
export const processDocument = action({
  args: { documentId: v.id("documents") },
  handler: async (ctx, { documentId }) => {
    // Schedule runs even if fetch below fails
    await ctx.scheduler.runAfter(0, internal.documents.markStarted, {
      documentId,
    });

    const result = await fetch("https://api.example.com/process");
    // If this throws, the scheduled function above still runs

    await ctx.runMutation(internal.documents.saveResult, {
      documentId,
      result: await result.json(),
    });
  },
});
```

### Schedule Immediately Pattern

Using `runAfter(0, ...)` adds a function to the queue immediately. This is useful for triggering actions from mutations conditionally on the mutation succeeding:

```typescript
export const requestSummary = mutation({
  args: { documentId: v.id("documents") },
  handler: async (ctx, { documentId }) => {
    await ctx.db.patch(documentId, { status: "processing" });

    // Action runs only if mutation commits successfully
    await ctx.scheduler.runAfter(0, internal.ai.generateSummary, {
      documentId,
    });
  },
});
```

### Auth is NOT Propagated

**Important**: Authentication is not propagated from the scheduling function to the scheduled function. Pass user information explicitly:

```typescript
export const scheduleUserTask = mutation({
  args: { taskData: v.string() },
  handler: async (ctx, { taskData }) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");

    // Pass user info explicitly
    await ctx.scheduler.runAfter(0, internal.tasks.process, {
      taskData,
      userId: identity.subject,
      userEmail: identity.email,
    });
  },
});
```

### Cancel Scheduled Functions

```typescript
export const sendReminder = mutation({
  args: { userId: v.id("users"), delay: v.number() },
  handler: async (ctx, { userId, delay }) => {
    // runAfter/runAt return a scheduled function ID
    const scheduledId = await ctx.scheduler.runAfter(
      delay,
      internal.reminders.send,
      { userId },
    );

    // Store the ID to cancel later
    await ctx.db.insert("scheduledReminders", { userId, scheduledId });
    return scheduledId;
  },
});

export const cancelReminder = mutation({
  args: { scheduledId: v.id("_scheduled_functions") },
  handler: async (ctx, { scheduledId }) => {
    await ctx.scheduler.cancel(scheduledId);
  },
});
```

**Cancel behavior**:

- If function hasn't started → it won't run
- If function already started → it continues, but any functions it schedules won't run

### Query Scheduled Functions

Every scheduled function is stored in the `_scheduled_functions` system table:

```typescript
export const getScheduledJobs = query({
  args: {},
  handler: async (ctx) => {
    return await ctx.db.system.query("_scheduled_functions").collect();
  },
});

export const getScheduledJob = query({
  args: { id: v.id("_scheduled_functions") },
  handler: async (ctx, { id }) => {
    return await ctx.db.system.get(id);
  },
});
```

**Document structure**:

```typescript
{
  _id: Id<"_scheduled_functions">,
  _creationTime: number,
  name: string,              // Function path, e.g., "messages.js:destruct"
  args: any[],               // Arguments passed to the function
  scheduledTime: number,     // When scheduled to run (ms since epoch)
  completedTime?: number,    // When finished (if completed)
  state: {
    kind: "pending" | "inProgress" | "success" | "failed" | "canceled"
  }
}
```

**States**:

| State        | Description                                                        |
| ------------ | ------------------------------------------------------------------ |
| `pending`    | Not started yet                                                    |
| `inProgress` | Running (actions only)                                             |
| `success`    | Completed successfully                                             |
| `failed`     | Hit an error (user or server)                                      |
| `canceled`   | Canceled via dashboard, `ctx.scheduler.cancel`, or parent canceled |

## Cron Jobs

Define recurring jobs in `convex/crons.ts`.

```typescript
// convex/crons.ts
import { cronJobs } from "convex/server";
import { internal } from "./_generated/api";

const crons = cronJobs();

// Interval-based (first run on deploy)
crons.interval(
  "cleanup-expired-sessions",
  { hours: 1 },
  internal.sessions.cleanupExpired,
);

// Time-based schedules (all times UTC)
crons.hourly(
  "sync-metrics",
  { minuteUTC: 0 }, // Top of every hour
  internal.metrics.sync,
);

crons.daily(
  "send-daily-digest",
  { hourUTC: 9, minuteUTC: 0 }, // 9:00 AM UTC
  internal.emails.sendDailyDigest,
);

crons.weekly(
  "weekly-report",
  { dayOfWeek: "monday", hourUTC: 8, minuteUTC: 0 },
  internal.reports.generateWeekly,
);

crons.monthly(
  "monthly-billing",
  { day: 1, hourUTC: 0, minuteUTC: 0 }, // 1st of month
  internal.billing.processMonthly,
);

// Standard cron expression syntax (UTC)
crons.cron("every-15-minutes", "*/15 * * * *", internal.jobs.checkHealth);

export default crons;
```

### Cron Schedule Methods

| Method       | Description                   | Example                                             |
| ------------ | ----------------------------- | --------------------------------------------------- |
| `interval()` | Every N seconds/minutes/hours | `{ minutes: 30 }`                                   |
| `hourly()`   | Every hour at minute          | `{ minuteUTC: 15 }`                                 |
| `daily()`    | Every day at time             | `{ hourUTC: 9, minuteUTC: 0 }`                      |
| `weekly()`   | Every week on day at time     | `{ dayOfWeek: 'monday', hourUTC: 8, minuteUTC: 0 }` |
| `monthly()`  | Every month on day at time    | `{ day: 1, hourUTC: 0, minuteUTC: 0 }`              |
| `cron()`     | Standard cron expression      | `'0 9 * * *'` (9 AM daily)                          |

**Note**: `interval()` allows seconds-level granularity. First run occurs on deploy.

### Cron with Arguments

```typescript
crons.interval(
  "cleanup-old-messages",
  { hours: 6 },
  internal.messages.cleanupOld,
  { maxAgeMs: 7 * 24 * 60 * 60 * 1000 }, // Static arguments
);
```

### Cron Execution Rules

- **At most one run at a time**: If a cron job is still running when the next scheduled time arrives, the next run is skipped
- **Skipped runs are logged**: Visible in the dashboard logs
- **All times in UTC**: Use [Crontab Guru](https://crontab.guru/) for cron expressions

## Error Handling

### Mutations: Exactly Once

Scheduled mutations are guaranteed to execute **exactly once**:

- Convex automatically retries on internal errors
- Only fails on developer errors (thrown exceptions)

### Actions: At Most Once

Scheduled actions execute **at most once**:

- Not automatically retried on transient errors
- May fail permanently on network issues, timeouts, etc.

**Manual retry pattern** for actions:

```typescript
export const processWithRetry = internalAction({
  args: {
    documentId: v.id("documents"),
    attempt: v.optional(v.number()),
  },
  handler: async (ctx, { documentId, attempt = 1 }) => {
    const maxAttempts = 3;

    try {
      const result = await fetch("https://api.example.com/process");
      if (!result.ok) throw new Error(`HTTP ${result.status}`);

      await ctx.runMutation(internal.documents.saveResult, {
        documentId,
        result: await result.json(),
      });
    } catch (error) {
      if (attempt < maxAttempts) {
        // Exponential backoff: 1s, 2s, 4s
        const delay = Math.pow(2, attempt - 1) * 1000;
        await ctx.scheduler.runAfter(
          delay,
          internal.processing.processWithRetry,
          { documentId, attempt: attempt + 1 },
        );
      } else {
        await ctx.runMutation(internal.documents.markFailed, {
          documentId,
          error: String(error),
        });
      }
    }
  },
});
```

## Workflow Patterns

### Action → Mutation Chain

```typescript
// Trigger workflow from mutation
export const startProcessing = mutation({
  args: { documentId: v.id("documents") },
  handler: async (ctx, { documentId }) => {
    await ctx.db.patch(documentId, {
      status: "processing",
      startedAt: Date.now(),
    });

    // Schedule action (runs after mutation commits)
    await ctx.scheduler.runAfter(0, internal.processing.processDocument, {
      documentId,
    });
  },
});

// Action does external work
export const processDocument = internalAction({
  args: { documentId: v.id("documents") },
  handler: async (ctx, { documentId }) => {
    try {
      const doc = await ctx.runQuery(internal.documents.get, { documentId });

      const result = await fetch("https://api.ai.com/process", {
        method: "POST",
        body: JSON.stringify({ content: doc.content }),
      });

      await ctx.runMutation(internal.processing.saveResult, {
        documentId,
        result: await result.json(),
      });
    } catch (error) {
      await ctx.runMutation(internal.processing.markFailed, {
        documentId,
        error: String(error),
      });
    }
  },
});
```

### Progress Tracking

```typescript
// Schema for job tracking
jobs: defineTable({
  type: v.string(),
  status: v.union(
    v.literal("pending"),
    v.literal("running"),
    v.literal("completed"),
    v.literal("failed"),
  ),
  progress: v.number(), // 0-100
  totalItems: v.optional(v.number()),
  processedItems: v.optional(v.number()),
  error: v.optional(v.string()),
  startedAt: v.optional(v.number()),
  completedAt: v.optional(v.number()),
});

// Update progress during batch processing
export const processBatch = internalAction({
  args: { jobId: v.id("jobs") },
  handler: async (ctx, { jobId }) => {
    const items = await ctx.runQuery(internal.items.listPending, {});

    await ctx.runMutation(internal.jobs.update, {
      jobId,
      status: "running",
      totalItems: items.length,
      processedItems: 0,
    });

    for (let i = 0; i < items.length; i++) {
      await processItem(items[i]);

      // Update progress every 10 items
      if (i % 10 === 0) {
        await ctx.runMutation(internal.jobs.update, {
          jobId,
          processedItems: i + 1,
          progress: Math.round(((i + 1) / items.length) * 100),
        });
      }
    }

    await ctx.runMutation(internal.jobs.update, {
      jobId,
      status: "completed",
      processedItems: items.length,
      progress: 100,
      completedAt: Date.now(),
    });
  },
});
```

## Limits & Guarantees

| Limit                         | Value                   |
| ----------------------------- | ----------------------- |
| Functions per scheduling call | 1000 max                |
| Total argument size           | 8MB max                 |
| Result retention              | 7 days after completion |
| Cron timezone                 | UTC only                |

**Execution guarantees**:

- Mutations: exactly once (auto-retry on internal errors)
- Actions: at most once (no auto-retry)
- Crons: at most one concurrent run per job

## Debugging

- **Dashboard Logs**: View executed scheduled function logs
- **Dashboard Functions**: View and cancel pending scheduled functions
- **Dashboard Schedules**: View cron job definitions and history

---

## Cron Expression Reference

```
┌───────────── minute (0-59)
│ ┌───────────── hour (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sunday=0)
│ │ │ │ │
* * * * *
```

### Common Cron Patterns

- `* * * * *` - Every minute
- `0 * * * *` - Every hour
- `0 0 * * *` - Every day at midnight
- `0 0 * * 0` - Every Sunday at midnight
- `0 0 1 * *` - First day of every month
- `*/5 * * * *` - Every 5 minutes
- `0 9-17 * * 1-5` - Every hour from 9 AM to 5 PM, Monday through Friday

---

## Monitoring and Logging for Cron Jobs

Add logging to track cron job execution:

```typescript
// convex/tasks.ts
import { internalMutation } from "./_generated/server";
import { v } from "convex/values";

export const cleanupWithLogging = internalMutation({
  args: {},
  returns: v.null(),
  handler: async (ctx) => {
    const startTime = Date.now();
    let processedCount = 0;
    let errorCount = 0;

    try {
      const expiredItems = await ctx.db
        .query("items")
        .withIndex("by_expiresAt")
        .filter((q) => q.lt(q.field("expiresAt"), Date.now()))
        .collect();

      for (const item of expiredItems) {
        try {
          await ctx.db.delete(item._id);
          processedCount++;
        } catch (error) {
          errorCount++;
          console.error(`Failed to delete item ${item._id}:`, error);
        }
      }

      // Log job completion
      await ctx.db.insert("cronLogs", {
        jobName: "cleanup",
        startTime,
        endTime: Date.now(),
        duration: Date.now() - startTime,
        processedCount,
        errorCount,
        status: errorCount === 0 ? "success" : "partial",
      });
    } catch (error) {
      // Log job failure
      await ctx.db.insert("cronLogs", {
        jobName: "cleanup",
        startTime,
        endTime: Date.now(),
        duration: Date.now() - startTime,
        processedCount,
        errorCount,
        status: "failed",
        error: String(error),
      });
      throw error;
    }

    return null;
  },
});
```

### Schema for Cron Job Logging

```typescript
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  cronLogs: defineTable({
    jobName: v.string(),
    startTime: v.number(),
    endTime: v.number(),
    duration: v.number(),
    processedCount: v.number(),
    errorCount: v.number(),
    status: v.union(
      v.literal("success"),
      v.literal("partial"),
      v.literal("failed"),
    ),
    error: v.optional(v.string()),
  })
    .index("by_job", ["jobName"])
    .index("by_status", ["status"])
    .index("by_startTime", ["startTime"]),
});
```

---

## Batching for Large Datasets in Crons

Handle large datasets in batches to avoid timeouts:

```typescript
// convex/tasks.ts
import { internalMutation } from "./_generated/server";
import { internal } from "./_generated/api";
import { v } from "convex/values";

const BATCH_SIZE = 100;

export const processBatch = internalMutation({
  args: {
    cursor: v.optional(v.string()),
  },
  returns: v.null(),
  handler: async (ctx, args) => {
    const result = await ctx.db
      .query("items")
      .withIndex("by_status", (q) => q.eq("status", "pending"))
      .paginate({ numItems: BATCH_SIZE, cursor: args.cursor ?? null });

    for (const item of result.page) {
      await ctx.db.patch(item._id, {
        status: "processed",
        processedAt: Date.now(),
      });
    }

    // Schedule next batch if there are more items
    if (!result.isDone) {
      await ctx.scheduler.runAfter(0, internal.tasks.processBatch, {
        cursor: result.continueCursor,
      });
    }

    return null;
  },
});
```

---

## External API Calls from Crons

Use actions for external API calls from cron jobs:

```typescript
// convex/sync.ts
"use node";

import { internalAction } from "./_generated/server";
import { internal } from "./_generated/api";
import { v } from "convex/values";

export const syncExternalData = internalAction({
  args: {},
  returns: v.null(),
  handler: async (ctx) => {
    // Fetch from external API
    const response = await fetch("https://api.example.com/data", {
      headers: {
        Authorization: `Bearer ${process.env.API_KEY}`,
      },
    });

    if (!response.ok) {
      throw new Error(`API request failed: ${response.status}`);
    }

    const data = await response.json();

    // Store the data using a mutation
    await ctx.runMutation(internal.sync.storeExternalData, {
      data,
      syncedAt: Date.now(),
    });

    return null;
  },
});

export const storeExternalData = internalMutation({
  args: {
    data: v.any(),
    syncedAt: v.number(),
  },
  returns: v.null(),
  handler: async (ctx, args) => {
    await ctx.db.insert("externalData", {
      data: args.data,
      syncedAt: args.syncedAt,
    });
    return null;
  },
});
```

```typescript
// convex/crons.ts — scheduling external API sync
import { cronJobs } from "convex/server";
import { internal } from "./_generated/api";

const crons = cronJobs();

crons.interval(
  "sync external data",
  { minutes: 15 },
  internal.sync.syncExternalData,
  {},
);

export default crons;
```

---

## Complete Cron Configuration Example

```typescript
// convex/crons.ts
import { cronJobs } from "convex/server";
import { internal } from "./_generated/api";

const crons = cronJobs();

// Cleanup jobs
crons.interval(
  "cleanup expired sessions",
  { hours: 1 },
  internal.cleanup.expiredSessions,
  {},
);

crons.interval("cleanup old logs", { hours: 24 }, internal.cleanup.oldLogs, {
  maxAgeDays: 30,
});

// Sync jobs
crons.interval("sync user data", { minutes: 15 }, internal.sync.userData, {});

// Report jobs
crons.cron("daily analytics", "0 1 * * *", internal.reports.dailyAnalytics, {});

crons.cron("weekly summary", "0 9 * * 1", internal.reports.weeklySummary, {});

// Health checks
crons.interval(
  "service health check",
  { minutes: 5 },
  internal.monitoring.healthCheck,
  {},
);

export default crons;
```
