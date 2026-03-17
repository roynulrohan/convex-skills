# Aggregate Component

Efficient O(log n) count, sum, min, max operations using denormalized data structures.

## Installation

```bash
npm install @convex-dev/aggregate
```

```typescript
// convex/convex.config.ts
import aggregate from "@convex-dev/aggregate/convex.config";

app.use(aggregate);
// Multiple aggregates need separate instances
app.use(aggregate, { name: "aggregateByScore" });
app.use(aggregate, { name: "aggregateByUser" });
```

## Setup

```typescript
// convex/aggregates.ts
import { TableAggregate } from "@convex-dev/aggregate";
import { components } from "./_generated/api";
import { DataModel } from "./_generated/dataModel";

// Simple count aggregate
export const scoreAggregate = new TableAggregate<{
  Key: number;
  DataModel: DataModel;
  TableName: "scores";
}>(components.aggregate, {
  sortKey: (doc) => doc.score,
});

// Namespaced aggregate (partition by game)
export const scoreByGame = new TableAggregate<{
  Namespace: string;
  Key: number;
  DataModel: DataModel;
  TableName: "scores";
}>(components.aggregateByScore, {
  namespace: (doc) => doc.gameId,
  sortKey: (doc) => doc.score,
  sumValue: (doc) => doc.score, // Enable sum operations
});
```

## Keeping Aggregate in Sync

### Manual Sync

```typescript
export const addScore = mutation({
  args: { playerId: v.string(), score: v.number(), gameId: v.string() },
  handler: async (ctx, args) => {
    const id = await ctx.db.insert("scores", args);
    const doc = await ctx.db.get(id);

    // Must manually sync
    await scoreAggregate.insert(ctx, doc!);
    await scoreByGame.insert(ctx, doc!);

    return id;
  },
});

export const updateScore = mutation({
  handler: async (ctx, { id, newScore }) => {
    const oldDoc = await ctx.db.get(id);
    await ctx.db.patch(id, { score: newScore });
    const newDoc = await ctx.db.get(id);

    await scoreAggregate.replace(ctx, oldDoc!, newDoc!);
  },
});

export const deleteScore = mutation({
  handler: async (ctx, { id }) => {
    const doc = await ctx.db.get(id);
    await ctx.db.delete(id);

    await scoreAggregate.delete(ctx, doc!);
  },
});
```

### Automatic Sync with Triggers (Recommended)

```typescript
import { Triggers } from "convex-helpers/server/triggers";
import {
  customCtx,
  customMutation,
} from "convex-helpers/server/customFunctions";
import { mutation as rawMutation } from "./_generated/server";

const triggers = new Triggers<DataModel>();

// Register aggregate triggers
triggers.register("scores", scoreAggregate.trigger());
triggers.register("scores", scoreByGame.trigger());

// Wrap mutations to auto-sync
export const mutation = customMutation(rawMutation, customCtx(triggers.wrapDB));

// Now any insert/update/delete auto-syncs
export const addScore = mutation({
  handler: async (ctx, args) => {
    // Aggregate syncs automatically!
    return await ctx.db.insert("scores", args);
  },
});
```

## Querying Aggregates

### Count

```typescript
// Total count
const total = await scoreAggregate.count(ctx);

// Count in namespace
const gameCount = await scoreByGame.count(ctx, { namespace: gameId });

// Count in range (scores 90-100)
const highScores = await scoreAggregate.count(ctx, {
  lower: { key: 90, inclusive: true },
  upper: { key: 100, inclusive: true },
});
```

### Sum

```typescript
// Total sum (requires sumValue in config)
const totalPoints = await scoreByGame.sum(ctx);

// Sum for namespace
const gameTotal = await scoreByGame.sum(ctx, { namespace: gameId });
```

### Min / Max

```typescript
const highScore = await scoreAggregate.max(ctx);
const lowScore = await scoreAggregate.min(ctx);

// Per game
const gameHighScore = await scoreByGame.max(ctx, { namespace: gameId });
```

### Offset-Based Pagination

```typescript
// Get 10 items starting at offset 20
const page = await scoreAggregate.paginate(ctx, {
  offset: 20,
  limit: 10,
});
```

### Rank Lookup

```typescript
// Find rank of specific score
const rank = await scoreAggregate.indexOf(ctx, { key: playerScore });
```

### Random Selection

```typescript
// Random document (useful for shuffling)
const randomDoc = await scoreAggregate.at(
  ctx,
  Math.floor(Math.random() * (await scoreAggregate.count(ctx))),
);
```

## Batch Operations

```typescript
// Efficient batch count
const counts = await scoreByGame.countBatch(ctx, [
  { namespace: "game1" },
  { namespace: "game2" },
  { namespace: "game3" },
]);

// Efficient batch sum
const sums = await scoreByGame.sumBatch(ctx, [
  { namespace: "game1" },
  { namespace: "game2" },
]);
```

## Backfilling Existing Data

If table has data before adding aggregate:

```typescript
import { Migrations } from "@convex-dev/migrations";

const migrations = new Migrations(components.migrations);

export const backfillAggregate = migrations.define({
  table: "scores",
  batchSize: 100,
  migrateOne: async (ctx, doc) => {
    await scoreAggregate.insertIfDoesNotExist(ctx, doc);
  },
});

// Run from dashboard or cron
export const runBackfill = migrations.runner(
  internal.migrations.backfillAggregate,
);
```

## Without Sorting (Random Order)

```typescript
const randomize = new TableAggregate<{
  Key: null;
  DataModel: DataModel;
  TableName: "items";
}>(components.aggregate, {
  sortKey: () => null, // No sorting = random order by _id
});

// Pick random item
const randomItem = await randomize.at(
  ctx,
  Math.floor(Math.random() * (await randomize.count(ctx))),
);
```

## DirectAggregate (Non-Table Data)

For aggregating data not stored in a Convex table, use `DirectAggregate`:

```typescript
import { DirectAggregate } from "@convex-dev/aggregate";

const tagCounts = new DirectAggregate<{
  Key: string;
  Id: string;
}>(components.aggregate, {});

// Insert arbitrary key-value pairs
await tagCounts.insert(ctx, { key: "typescript", id: "post-1" });
await tagCounts.insert(ctx, { key: "react", id: "post-2" });

// Same query API as TableAggregate
const total = await tagCounts.count(ctx);
```

Unlike `TableAggregate` (which derives keys from documents), `DirectAggregate` takes explicit `{ key, id, sumValue? }` objects.

## Idempotent Write Methods

Safe methods for use during backfill or when writes may overlap:

```typescript
// Insert only if this document isn't already tracked
await scoreAggregate.insertIfDoesNotExist(ctx, doc);

// Insert or update if already exists
await scoreAggregate.replaceOrInsert(ctx, oldDoc, newDoc);

// Delete only if tracked (no error if missing)
await scoreAggregate.deleteIfExists(ctx, doc);
```

For triggers during backfill, use `idempotentTrigger()` instead of `trigger()`:

```typescript
// During backfill period - tolerates duplicates
triggers.register("scores", scoreAggregate.idempotentTrigger());

// After backfill completes - switch back to standard trigger
triggers.register("scores", scoreAggregate.trigger());
```

## Performance Notes

- **O(log n)** for all operations vs O(n) for naive `.collect()`
- Reactive: UI updates when aggregate changes
- Transactional: Aggregate always consistent with source table
- High-write tables may cause OCC conflicts - consider sharding or Sharded Counter for simple counts
