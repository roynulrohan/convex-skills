# Dynamic Crons Component

Runtime-configurable cron jobs — register, update, and delete schedules from mutations at runtime.

**How it differs from built-in `cronJobs()`:** Built-in crons are static and defined at deploy time. This component lets you create and manage cron schedules dynamically from mutations, with transactional guarantees.

## Installation

```bash
npm install @convex-dev/crons
```

```typescript
// convex/convex.config.ts
import crons from "@convex-dev/crons/convex.config.js";
app.use(crons);
```

## Setup

```typescript
// convex/crons.ts
import { Crons } from "@convex-dev/crons";
import { components } from "./_generated/api";

const crons = new Crons(components.crons);
```

## Registering Cron Jobs

### Cron Schedule (Unix Crontab Syntax)

```typescript
import { internalMutation } from "./_generated/server";
import { internal } from "./_generated/api";

export const registerDailyReport = internalMutation({
  handler: async (ctx) => {
    const cronId = await crons.register(
      ctx,
      { kind: "cron", cronspec: "0 0 * * *" }, // Daily at midnight
      internal.reports.generateDaily,
      { type: "summary" }, // Args for the function
      "daily-report", // Optional name
    );
    return cronId;
  },
});
```

### Interval Schedule

```typescript
export const registerHealthCheck = internalMutation({
  handler: async (ctx) => {
    const cronId = await crons.register(
      ctx,
      { kind: "interval", ms: 60000 }, // Every 60 seconds
      internal.monitoring.healthCheck,
      {},
    );
    return cronId;
  },
});
```

### Schedule Format

Two schedule types:

```typescript
// Cron expression (6-field Unix crontab, optional timezone)
{ kind: 'cron', cronspec: '0 0 * * *' }
{ kind: 'cron', cronspec: '0 9 * * 1', tz: 'America/New_York' }

// Fixed interval (minimum 1000ms)
{ kind: 'interval', ms: 3600000 }
```

**Cron expression fields:**

```
 *  *  *  *  *  *
 │  │  │  │  │  └─ day of week (0-7, 0 or 7 = Sunday, 1L-7L for "last X of month")
 │  │  │  │  └──── month (1-12)
 │  │  │  └─────── day of month (1-31, L for last day)
 │  │  └────────── hour (0-23)
 │  └───────────── minute (0-59)
 └──────────────── second (0-59, optional)
```

**Examples:**
| Expression | Meaning |
|------------|---------|
| `0 0 * * *` | Daily at midnight |
| `*/15 * * * *` | Every 15 minutes |
| `0 9 * * 1` | Every Monday at 9am |
| `0 0 1 * *` | First day of every month |
| `30 0 * * * *` | Every minute at 30 seconds |

## Managing Cron Jobs

### Get by Name or ID

```typescript
// By name
const cron = await crons.get(ctx, { name: "daily-report" });

// By ID
const cron = await crons.get(ctx, { id: cronId });

// Returns cron job object or null
```

### List All

```typescript
const allCrons = await crons.list(ctx);
```

### Delete

```typescript
// By name
await crons.delete(ctx, { name: "daily-report" });

// By ID
await crons.delete(ctx, { id: cronId });
```

## API Reference

| Method                                     | Args                                           | Returns            | Description                      |
| ------------------------------------------ | ---------------------------------------------- | ------------------ | -------------------------------- |
| `register(ctx, schedule, fn, args, name?)` | Schedule + function ref + args + optional name | `string` (cron ID) | Create a new cron job            |
| `get(ctx, { name } \| { id })`             | Name or ID                                     | `CronInfo \| null` | Look up a cron job               |
| `list(ctx)`                                | —                                              | `CronInfo[]`       | List all registered cron jobs    |
| `delete(ctx, { name } \| { id })`          | Name or ID                                     | `null`             | Delete and deschedule a cron job |

### CronInfo Type

```typescript
type CronInfo = {
  id: string;
  name?: string;
  functionHandle: FunctionHandle<"mutation" | "action">;
  args: Record<string, unknown>;
  schedule: Schedule;
};
```

### Validation

- **Interval**: `ms` must be >= 1000, otherwise throws
- **Cron**: `cronspec` is validated via `cron-parser`; invalid expressions throw
- **Names**: Must be unique — throws if a cron with the same name already exists

## Common Patterns

### Idempotent Registration

Prevent duplicate crons by checking existence first:

```typescript
export const ensureDailyCron = internalMutation({
  handler: async (ctx) => {
    const existing = await crons.get(ctx, { name: "daily-report" });
    if (existing !== null) return; // Already registered

    await crons.register(
      ctx,
      { kind: "cron", cronspec: "0 0 * * *" },
      internal.reports.generateDaily,
      {},
      "daily-report",
    );
  },
});
```

### Self-Deleting Cron

A cron that runs once and cleans itself up:

```typescript
export const registerOneShot = internalMutation({
  handler: async (ctx) => {
    await crons.register(
      ctx,
      { kind: "interval", ms: 10000 },
      internal.tasks.runAndDelete,
      { name: "one-shot-task" },
      "one-shot-task",
    );
  },
});

export const runAndDelete = internalMutation({
  args: { name: v.string() },
  handler: async (ctx, { name }) => {
    // Do the work
    await doSomething(ctx);
    // Delete self
    await crons.delete(ctx, { name });
  },
});
```

### User-Configurable Schedules

Let users set their own report schedule:

```typescript
export const setReportSchedule = mutation({
  args: { cronspec: v.string() },
  handler: async (ctx, { cronspec }) => {
    const userId = await getAuthUserId(ctx);

    // Delete existing schedule
    const existing = await crons.get(ctx, { name: `report-${userId}` });
    if (existing) {
      await crons.delete(ctx, { name: `report-${userId}` });
    }

    // Register new schedule
    await crons.register(
      ctx,
      { kind: "cron", cronspec },
      internal.reports.generateForUser,
      { userId },
      `report-${userId}`,
    );
  },
});
```

### Update a Cron Schedule

Delete and re-register (there's no update method):

```typescript
export const updateSchedule = internalMutation({
  args: { name: v.string(), newCronspec: v.string() },
  handler: async (ctx, { name, newCronspec }) => {
    const existing = await crons.get(ctx, { name });
    if (!existing) throw new Error(`Cron "${name}" not found`);

    await crons.delete(ctx, { name });
    await crons.register(
      ctx,
      { kind: "cron", cronspec: newCronspec },
      existing.fn,
      existing.args,
      name,
    );
  },
});
```

## Key Properties

- **Transactional**: Crons are guaranteed to exist after the mutation that created them commits
- **Runtime**: No redeployment needed to add/change/remove schedules
- **Named**: Optional names enable idempotent registration and easy lookup
- **No overlap**: If a previous execution is still running, the next scheduled run is skipped
- **Isolated execution**: Target function runs in a separate transaction from the scheduler — failures don't break the cron loop
- **Timezone support**: Optional `tz` field on cron schedules (IANA timezone strings)
- **Seconds precision**: Optional 6th cron field for second-level scheduling

## When to Use This vs Built-in Crons

| Feature            | Built-in `cronJobs()` | `@convex-dev/crons`      |
| ------------------ | --------------------- | ------------------------ |
| Definition time    | Static, at deploy     | Dynamic, at runtime      |
| Modification       | Requires redeploy     | Create/delete at runtime |
| Self-deleting jobs | Not possible          | Supported                |
| Timezone support   | Yes                   | Yes (`tz` field)         |
| Seconds precision  | No                    | Yes (optional 6th field) |
| Minimum interval   | 1 minute              | 1 second (1000ms)        |
| Overlap prevention | Built-in              | Built-in                 |
| Transactional      | N/A                   | Yes                      |

**Rule of thumb:** Use built-in `cronJobs()` for fixed schedules known at deploy time. Use this component when schedules are determined by user input or need to change without redeploying.
