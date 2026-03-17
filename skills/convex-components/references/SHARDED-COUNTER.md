# Sharded Counter

High-throughput counting component that spreads counter updates across multiple shard documents to reduce write conflicts. Ideal for frequently updated values like view counts, like counts, or inventory tracking.

## When to Use

- **Sharded Counter**: When you need a total count/sum and the counter is updated frequently (high write throughput)
- **Aggregate**: When you need ordering, ranking, or pagination by a computed key (leaderboards)

Rule of thumb: **Totals → sharded counters**, **Ranked lists → aggregates**

## Installation

```bash
npm install @convex-dev/sharded-counter
```

## Setup

### Register in `convex.config.ts`

```typescript
// convex/convex.config.ts
import { defineApp } from "convex/server";
import shardedCounter from "@convex-dev/sharded-counter/convex.config";

const app = defineApp();
app.use(shardedCounter);

export default app;
```

### Create an Instance

```typescript
// convex/counter.ts
import { components } from "./_generated/api";
import { ShardedCounter } from "@convex-dev/sharded-counter";

const counter = new ShardedCounter(components.shardedCounter);
```

## API Reference

### Writing (Mutations/Actions)

```typescript
// Direct key usage
await counter.add(ctx, "checkboxes", 5); // increment by 5
await counter.inc(ctx, "checkboxes"); // increment by 1
await counter.subtract(ctx, "checkboxes", 5); // decrement by 5
await counter.dec(ctx, "checkboxes"); // decrement by 1
await counter.reset(ctx, "checkboxes"); // reset to 0

// Key-specific instance (useful when operating on same key repeatedly)
const numCheckboxes = counter.for("checkboxes");
await numCheckboxes.inc(ctx); // increment by 1
await numCheckboxes.dec(ctx); // decrement by 1
await numCheckboxes.add(ctx, 5); // add 5
await numCheckboxes.subtract(ctx, 5); // subtract 5
await numCheckboxes.reset(ctx); // reset to 0
```

### Reading (Queries/Mutations)

```typescript
// Get exact count
const count = await counter.count(ctx, "checkboxes");

// Or via key-specific instance
const numCheckboxes = counter.for("checkboxes");
const count = await numCheckboxes.count(ctx);
```

## Complete Example

```typescript
// convex/likes.ts
import { mutation, query } from "./_generated/server";
import { v } from "convex/values";
import { components } from "./_generated/api";
import { ShardedCounter } from "@convex-dev/sharded-counter";

const counter = new ShardedCounter(components.shardedCounter);

export const like = mutation({
  args: { postId: v.id("posts") },
  handler: async (ctx, { postId }) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");

    // Check if already liked
    const existing = await ctx.db
      .query("likes")
      .withIndex("by_post_user", (q) =>
        q.eq("postId", postId).eq("userId", identity.subject),
      )
      .unique();

    if (existing) return;

    await ctx.db.insert("likes", {
      postId,
      userId: identity.subject,
    });

    // Increment counter (high throughput, no OCC conflicts)
    await counter.inc(ctx, `post:${postId}:likes`);
  },
});

export const unlike = mutation({
  args: { postId: v.id("posts") },
  handler: async (ctx, { postId }) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");

    const existing = await ctx.db
      .query("likes")
      .withIndex("by_post_user", (q) =>
        q.eq("postId", postId).eq("userId", identity.subject),
      )
      .unique();

    if (!existing) return;

    await ctx.db.delete(existing._id);
    await counter.dec(ctx, `post:${postId}:likes`);
  },
});

export const getLikeCount = query({
  args: { postId: v.id("posts") },
  handler: async (ctx, { postId }) => {
    return await counter.count(ctx, `post:${postId}:likes`);
  },
});
```

## Multiple Named Instances

If you need separate counter namespaces:

```typescript
// convex/convex.config.ts
app.use(shardedCounter, { name: "likesCounter" });
app.use(shardedCounter, { name: "viewsCounter" });
```

```typescript
const likes = new ShardedCounter(components.likesCounter);
const views = new ShardedCounter(components.viewsCounter);
```

## Key Design Tips

- **Use descriptive string keys**: `post:${postId}:likes`, `user:${userId}:points`
- **Counters don't enforce uniqueness**: If you need "distinct users liked", track membership in a separate table and use the counter just for the count
- **Counters are reactive**: `counter.count()` in a query will update in real-time
- **No OCC conflicts**: Unlike reading + patching a single document, sharded counters spread writes across shards so concurrent increments don't conflict

## Troubleshooting

**How does sharded counter improve performance over regular Convex counters?**

It splits counter state across multiple database documents instead of storing everything in a single document. This eliminates write conflicts when multiple users increment the counter simultaneously, allowing much higher throughput than traditional single-document counters.

**Is the sharded counter eventually consistent or strongly consistent?**

Strong consistency for individual increment operations and eventually consistent reads. Each increment operation is atomic within its shard, and read operations aggregate all shards to return the current total value.

**How many concurrent operations can the sharded counter handle?**

Automatically scales based on contention levels and can handle thousands of concurrent increment operations. The component dynamically manages shard allocation to distribute load and prevent any single shard from becoming a bottleneck.

**Can I use sharded counters for decrementing values?**

Yes — both increment and decrement operations are distributed across shards in the same way, allowing you to build features like upvote/downvote systems or inventory tracking with high concurrency.

## Resources

- [npm package](https://www.npmjs.com/package/@convex-dev/sharded-counter)
- [GitHub repository](https://github.com/get-convex/sharded-counter)
- [Convex Components Directory](https://www.convex.dev/components/sharded-counter)
