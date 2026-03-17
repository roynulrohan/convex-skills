---
name: convex-security
description: "Security patterns for Convex applications including authentication checks, authorization (RBAC, permissions), row-level access control, argument validation, function exposure review, action isolation, rate limiting, audit trails, and environment variable handling. Use when reviewing Convex code for security, implementing auth patterns, adding rate limiting, building RBAC, or auditing data access boundaries."
---

# Convex Security

Comprehensive security patterns for Convex applications including authentication, authorization (RBAC, permissions), row-level access control, argument validation, function exposure review, action isolation, rate limiting, audit trails, and environment variable handling.

## Documentation Sources

Before implementing, do not assume; fetch the latest documentation:

- Primary: https://docs.convex.dev/auth
- Functions Auth: https://docs.convex.dev/auth/functions-auth
- Production Security: https://docs.convex.dev/production
- For broader context: https://docs.convex.dev/llms.txt

## Instructions

### Quick Security Checklist

Use this checklist to quickly audit your Convex application's security:

#### 1. Authentication

- [ ] Authentication provider configured (Clerk, Auth0, etc.)
- [ ] All sensitive queries check `ctx.auth.getUserIdentity()`
- [ ] Unauthenticated access explicitly allowed where intended
- [ ] Session tokens properly validated

#### 2. Function Exposure

- [ ] Public functions (`query`, `mutation`, `action`) reviewed
- [ ] Internal functions use `internalQuery`, `internalMutation`, `internalAction`
- [ ] No sensitive operations exposed as public functions
- [ ] HTTP actions validate origin/authentication

#### 3. Argument Validation

- [ ] All functions have explicit `args` validators
- [ ] All functions have explicit `returns` validators
- [ ] No `v.any()` used for sensitive data
- [ ] ID validators use correct table names

#### 4. Row-Level Access Control

- [ ] Users can only access their own data
- [ ] Admin functions check user roles
- [ ] Shared resources have proper access checks
- [ ] Deletion functions verify ownership

#### 5. Environment Variables

- [ ] API keys stored in environment variables
- [ ] No secrets in code or schema
- [ ] Different keys for dev/prod environments
- [ ] Environment variables accessed only in actions

### Authentication Patterns

#### Simple requireAuth Helper

```typescript
// convex/auth.ts
import { QueryCtx, MutationCtx } from "./_generated/server";
import { query } from "./_generated/server";
import { v } from "convex/values";
import { ConvexError } from "convex/values";

// Simple authentication check
async function requireAuth(ctx: QueryCtx | MutationCtx) {
  const identity = await ctx.auth.getUserIdentity();
  if (!identity) {
    throw new ConvexError("Authentication required");
  }
  return identity;
}

// Secure query pattern
export const getMyProfile = query({
  args: {},
  returns: v.union(
    v.object({
      _id: v.id("users"),
      name: v.string(),
      email: v.string(),
    }),
    v.null(),
  ),
  handler: async (ctx) => {
    const identity = await requireAuth(ctx);

    return await ctx.db
      .query("users")
      .withIndex("by_tokenIdentifier", (q) =>
        q.eq("tokenIdentifier", identity.tokenIdentifier),
      )
      .unique();
  },
});
```

#### Full getUser / getAuthenticatedUser Helper

```typescript
// convex/lib/auth.ts
import { QueryCtx, MutationCtx } from "./_generated/server";
import { ConvexError } from "convex/values";
import { Doc } from "./_generated/dataModel";

export async function getUser(
  ctx: QueryCtx | MutationCtx,
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

export async function getAuthenticatedUser(ctx: QueryCtx | MutationCtx) {
  const identity = await ctx.auth.getUserIdentity();
  if (!identity) {
    throw new ConvexError({
      code: "UNAUTHENTICATED",
      message: "You must be logged in",
    });
  }

  const user = await ctx.db
    .query("users")
    .withIndex("by_tokenIdentifier", (q) =>
      q.eq("tokenIdentifier", identity.tokenIdentifier),
    )
    .unique();

  if (!user) {
    throw new ConvexError({
      code: "USER_NOT_FOUND",
      message: "User profile not found",
    });
  }

  return user;
}
```

### Authorization & RBAC

#### Role-Based Access Control

```typescript
// convex/lib/auth.ts (continued)
type UserRole = "user" | "moderator" | "admin" | "superadmin";

const roleHierarchy: Record<UserRole, number> = {
  user: 0,
  moderator: 1,
  admin: 2,
  superadmin: 3,
};

export async function requireRole(
  ctx: QueryCtx | MutationCtx,
  minRole: UserRole,
): Promise<Doc<"users">> {
  const user = await getUser(ctx);

  if (!user) {
    throw new ConvexError({
      code: "UNAUTHENTICATED",
      message: "Authentication required",
    });
  }

  const userRoleLevel = roleHierarchy[user.role as UserRole] ?? 0;
  const requiredLevel = roleHierarchy[minRole];

  if (userRoleLevel < requiredLevel) {
    throw new ConvexError({
      code: "FORBIDDEN",
      message: `Role '${minRole}' or higher required`,
    });
  }

  return user;
}
```

#### Permission-Based Access Control

```typescript
// convex/lib/auth.ts (continued)
type Permission =
  | "read:users"
  | "write:users"
  | "delete:users"
  | "admin:system";

const rolePermissions: Record<UserRole, Permission[]> = {
  user: ["read:users"],
  moderator: ["read:users", "write:users"],
  admin: ["read:users", "write:users", "delete:users"],
  superadmin: ["read:users", "write:users", "delete:users", "admin:system"],
};

export async function requirePermission(
  ctx: QueryCtx | MutationCtx,
  permission: Permission,
): Promise<Doc<"users">> {
  const user = await getUser(ctx);

  if (!user) {
    throw new ConvexError({
      code: "UNAUTHENTICATED",
      message: "Authentication required",
    });
  }

  const userRole = user.role as UserRole;
  const permissions = rolePermissions[userRole] ?? [];

  if (!permissions.includes(permission)) {
    throw new ConvexError({
      code: "FORBIDDEN",
      message: `Permission '${permission}' required`,
    });
  }

  return user;
}
```

#### Simple Admin Check

```typescript
// Simpler alternative when you only need user vs admin
async function requireAdmin(ctx: QueryCtx | MutationCtx) {
  const user = await getAuthenticatedUser(ctx);

  if (user.role !== "admin") {
    throw new ConvexError({
      code: "FORBIDDEN",
      message: "Admin access required",
    });
  }

  return user;
}
```

### Row-Level Access Control / Data Access Boundaries

#### Users Can Only See Their Own Data

```typescript
// convex/data.ts
import { query, mutation } from "./_generated/server";
import { v } from "convex/values";
import { getUser } from "./lib/auth";
import { ConvexError } from "convex/values";

export const getMyData = query({
  args: {},
  returns: v.array(
    v.object({
      _id: v.id("userData"),
      content: v.string(),
    }),
  ),
  handler: async (ctx) => {
    const user = await getUser(ctx);
    if (!user) return [];

    // SECURITY: Filter by userId
    return await ctx.db
      .query("userData")
      .withIndex("by_user", (q) => q.eq("userId", user._id))
      .collect();
  },
});
```

#### Verify Ownership Before Returning Sensitive Data

```typescript
export const getSensitiveItem = query({
  args: { itemId: v.id("sensitiveItems") },
  returns: v.union(
    v.object({
      _id: v.id("sensitiveItems"),
      secret: v.string(),
    }),
    v.null(),
  ),
  handler: async (ctx, args) => {
    const user = await getUser(ctx);
    if (!user) return null;

    const item = await ctx.db.get(args.itemId);

    // SECURITY: Verify ownership
    if (!item || item.ownerId !== user._id) {
      return null; // Don't reveal if item exists
    }

    return item;
  },
});
```

#### Verify Ownership Before Update/Delete

```typescript
export const updateTask = mutation({
  args: {
    taskId: v.id("tasks"),
    title: v.string(),
  },
  returns: v.null(),
  handler: async (ctx, args) => {
    const identity = await requireAuth(ctx);

    const task = await ctx.db.get(args.taskId);

    // Check ownership
    if (!task || task.userId !== identity.tokenIdentifier) {
      throw new ConvexError("Not authorized to update this task");
    }

    await ctx.db.patch(args.taskId, { title: args.title });
    return null;
  },
});

export const deleteTask = mutation({
  args: { taskId: v.id("tasks") },
  returns: v.null(),
  handler: async (ctx, args) => {
    const identity = await requireAuth(ctx);

    const task = await ctx.db.get(args.taskId);

    if (!task || task.userId !== identity.tokenIdentifier) {
      throw new ConvexError("Not authorized to delete this task");
    }

    await ctx.db.delete(args.taskId);
    return null;
  },
});
```

#### Shared Resources with Access Lists

```typescript
export const getSharedDocument = query({
  args: { docId: v.id("documents") },
  returns: v.union(
    v.object({
      _id: v.id("documents"),
      content: v.string(),
      accessLevel: v.string(),
    }),
    v.null(),
  ),
  handler: async (ctx, args) => {
    const user = await getUser(ctx);
    const doc = await ctx.db.get(args.docId);

    if (!doc) return null;

    // Public documents
    if (doc.visibility === "public") {
      return { ...doc, accessLevel: "public" };
    }

    // Must be authenticated for non-public
    if (!user) return null;

    // Owner has full access
    if (doc.ownerId === user._id) {
      return { ...doc, accessLevel: "owner" };
    }

    // Check shared access
    const access = await ctx.db
      .query("documentAccess")
      .withIndex("by_doc_and_user", (q) =>
        q.eq("documentId", args.docId).eq("userId", user._id),
      )
      .unique();

    if (!access) return null;

    return { ...doc, accessLevel: access.level };
  },
});
```

### Function Exposure

```typescript
// PUBLIC - Exposed to clients (review carefully!)
export const listPublicPosts = query({
  args: {},
  returns: v.array(
    v.object({
      /* ... */
    }),
  ),
  handler: async (ctx) => {
    // Anyone can call this - intentionally public
    return await ctx.db
      .query("posts")
      .withIndex("by_public", (q) => q.eq("isPublic", true))
      .collect();
  },
});

// INTERNAL - Only callable from other Convex functions
export const _updateUserCredits = internalMutation({
  args: { userId: v.id("users"), amount: v.number() },
  returns: v.null(),
  handler: async (ctx, args) => {
    // This cannot be called directly from clients
    await ctx.db.patch(args.userId, {
      credits: args.amount,
    });
    return null;
  },
});

// Internal action - not exposed to clients
export const _processPayment = internalAction({
  args: {
    userId: v.id("users"),
    amount: v.number(),
    paymentMethodId: v.string(),
  },
  returns: v.object({
    success: v.boolean(),
    transactionId: v.optional(v.string()),
  }),
  handler: async (ctx, args) => {
    const stripeKey = process.env.STRIPE_SECRET_KEY;

    // Process payment with Stripe
    // This should NEVER be exposed as a public action

    return { success: true, transactionId: "txn_xxx" };
  },
});
```

### Argument Validation

```typescript
// GOOD: Strict validation
export const createPost = mutation({
  args: {
    title: v.string(),
    content: v.string(),
    category: v.union(v.literal("tech"), v.literal("news"), v.literal("other")),
  },
  returns: v.id("posts"),
  handler: async (ctx, args) => {
    const identity = await requireAuth(ctx);
    return await ctx.db.insert("posts", {
      ...args,
      authorId: identity.tokenIdentifier,
    });
  },
});

// BAD: Weak validation
export const createPostUnsafe = mutation({
  args: {
    data: v.any(), // DANGEROUS: Allows any data
  },
  returns: v.id("posts"),
  handler: async (ctx, args) => {
    return await ctx.db.insert("posts", args.data);
  },
});
```

### Action Isolation

```typescript
// convex/actions.ts
"use node";

import { action, internalAction } from "./_generated/server";
import { v } from "convex/values";
import { api, internal } from "./_generated/api";
import { ConvexError } from "convex/values";

// SECURITY: Never expose API keys in responses
export const callExternalAPI = action({
  args: { query: v.string() },
  returns: v.object({ result: v.string() }),
  handler: async (ctx, args) => {
    // Verify user is authenticated
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) {
      throw new ConvexError("Authentication required");
    }

    // Get API key from environment (not hardcoded)
    const apiKey = process.env.EXTERNAL_API_KEY;
    if (!apiKey) {
      throw new Error("API key not configured");
    }

    // Log usage for audit trail
    await ctx.runMutation(internal.audit.logAPICall, {
      userId: identity.tokenIdentifier,
      endpoint: "external-api",
      timestamp: Date.now(),
    });

    const response = await fetch("https://api.example.com/query", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ query: args.query }),
    });

    if (!response.ok) {
      // Don't expose external API error details
      throw new ConvexError("External service unavailable");
    }

    const data = await response.json();

    // Sanitize response before returning
    return { result: sanitizeResponse(data) };
  },
});
```

### Rate Limiting

Use the `@convex-dev/rate-limiter` component for application-layer rate limiting. Do not hand-roll rate limiting with custom tables.

```typescript
// convex/rateLimiter.ts
import { RateLimiter, MINUTE, HOUR } from "@convex-dev/rate-limiter";
import { components } from "./_generated/api";

export const rateLimiter = new RateLimiter(components.rateLimiter, {
  sendMessage: {
    kind: "token bucket",
    rate: 10, // 10 per minute sustained
    period: MINUTE,
    capacity: 3, // Allow burst of 3
  },
  upload: { kind: "fixed window", rate: 5, period: MINUTE * 5 },
  apiCall: { kind: "token bucket", rate: 100, period: HOUR },
});
```

```typescript
// convex/messages.ts
import { mutation } from "./_generated/server";
import { v } from "convex/values";
import { ConvexError } from "convex/values";
import { rateLimiter } from "./rateLimiter";

export const sendMessage = mutation({
  args: { content: v.string() },
  returns: v.id("messages"),
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new ConvexError("Authentication required");

    // Check and consume rate limit (throws ConvexError if limited)
    await rateLimiter.limit(ctx, "sendMessage", {
      key: identity.tokenIdentifier,
      throws: true,
    });

    return await ctx.db.insert("messages", {
      content: args.content,
      authorId: identity.tokenIdentifier,
      createdAt: Date.now(),
    });
  },
});
```

Handle rate limit errors on the client:

```typescript
import { isRateLimitError } from "@convex-dev/rate-limiter";

try {
  await sendMessage({ content });
} catch (e) {
  if (isRateLimitError(e)) {
    toast.error(`Try again in ${Math.ceil(e.data.retryAfter / 1000)}s`);
  }
}
```

### Sensitive Operations Protection

```typescript
// convex/admin.ts
import { mutation, internalMutation } from "./_generated/server";
import { v } from "convex/values";
import { requireRole } from "./lib/auth";
import { internal } from "./_generated/api";

// Two-factor confirmation for dangerous operations
export const deleteAllUserData = mutation({
  args: {
    userId: v.id("users"),
    confirmationCode: v.string(),
  },
  returns: v.null(),
  handler: async (ctx, args) => {
    // Require superadmin
    const admin = await requireRole(ctx, "superadmin");

    // Verify confirmation code
    const confirmation = await ctx.db
      .query("confirmations")
      .withIndex("by_admin_and_code", (q) =>
        q.eq("adminId", admin._id).eq("code", args.confirmationCode),
      )
      .filter((q) => q.gt(q.field("expiresAt"), Date.now()))
      .unique();

    if (!confirmation || confirmation.action !== "delete_user_data") {
      throw new ConvexError("Invalid or expired confirmation code");
    }

    // Delete confirmation to prevent reuse
    await ctx.db.delete(confirmation._id);

    // Schedule deletion (don't do it inline)
    await ctx.scheduler.runAfter(0, internal.admin._performDeletion, {
      userId: args.userId,
      requestedBy: admin._id,
    });

    // Audit log
    await ctx.db.insert("auditLogs", {
      action: "delete_user_data",
      targetUserId: args.userId,
      performedBy: admin._id,
      timestamp: Date.now(),
    });

    return null;
  },
});

// Generate confirmation code for sensitive action
export const requestDeletionConfirmation = mutation({
  args: { userId: v.id("users") },
  returns: v.string(),
  handler: async (ctx, args) => {
    const admin = await requireRole(ctx, "superadmin");

    const code = generateSecureCode();

    await ctx.db.insert("confirmations", {
      adminId: admin._id,
      code,
      action: "delete_user_data",
      targetUserId: args.userId,
      expiresAt: Date.now() + 5 * 60 * 1000, // 5 minutes
    });

    // In production, send code via secure channel (email, SMS)
    return code;
  },
});
```

### Audit Trail System

```typescript
// convex/audit.ts
import { mutation, query, internalMutation } from "./_generated/server";
import { v } from "convex/values";
import { getUser, requireRole } from "./lib/auth";

const auditEventValidator = v.object({
  _id: v.id("auditLogs"),
  _creationTime: v.number(),
  action: v.string(),
  userId: v.optional(v.string()),
  resourceType: v.string(),
  resourceId: v.string(),
  details: v.optional(v.any()),
  ipAddress: v.optional(v.string()),
  timestamp: v.number(),
});

// Internal: Log audit event
export const logEvent = internalMutation({
  args: {
    action: v.string(),
    userId: v.optional(v.string()),
    resourceType: v.string(),
    resourceId: v.string(),
    details: v.optional(v.any()),
  },
  returns: v.id("auditLogs"),
  handler: async (ctx, args) => {
    return await ctx.db.insert("auditLogs", {
      ...args,
      timestamp: Date.now(),
    });
  },
});

// Admin: View audit logs
export const getAuditLogs = query({
  args: {
    resourceType: v.optional(v.string()),
    userId: v.optional(v.string()),
    limit: v.optional(v.number()),
  },
  returns: v.array(auditEventValidator),
  handler: async (ctx, args) => {
    await requireRole(ctx, "admin");

    let query = ctx.db.query("auditLogs");

    if (args.resourceType) {
      query = query.withIndex("by_resource_type", (q) =>
        q.eq("resourceType", args.resourceType),
      );
    }

    return await query.order("desc").take(args.limit ?? 100);
  },
});
```

### Environment Variables

```typescript
// convex/actions.ts
"use node";

import { action } from "./_generated/server";
import { v } from "convex/values";

export const sendEmail = action({
  args: {
    to: v.string(),
    subject: v.string(),
    body: v.string(),
  },
  returns: v.object({ success: v.boolean() }),
  handler: async (ctx, args) => {
    // Access API key from environment
    const apiKey = process.env.RESEND_API_KEY;

    if (!apiKey) {
      throw new Error("RESEND_API_KEY not configured");
    }

    const response = await fetch("https://api.resend.com/emails", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        from: "noreply@example.com",
        to: args.to,
        subject: args.subject,
        html: args.body,
      }),
    });

    return { success: response.ok };
  },
});
```

## Examples

### Putting It All Together

Using the helpers defined above in a real function:

```typescript
// convex/secure.ts
import { query, internalMutation } from "./_generated/server";
import { v } from "convex/values";
import { getAuthenticatedUser, requireRole } from "./lib/auth";

// Public: List own tasks (uses auth helper)
export const listMyTasks = query({
  args: {},
  returns: v.array(
    v.object({
      _id: v.id("tasks"),
      title: v.string(),
      completed: v.boolean(),
    }),
  ),
  handler: async (ctx) => {
    const user = await getAuthenticatedUser(ctx);
    return await ctx.db
      .query("tasks")
      .withIndex("by_user", (q) => q.eq("userId", user._id))
      .collect();
  },
});

// Admin only: List all users (uses role helper)
export const listAllUsers = query({
  args: {},
  handler: async (ctx) => {
    await requireRole(ctx, "admin");
    return await ctx.db.query("users").collect();
  },
});
```

## Best Practices & Common Pitfalls

| Do                                                        | Don't                                           |
| --------------------------------------------------------- | ----------------------------------------------- |
| Verify identity in all public functions                   | Assume client-provided IDs are safe             |
| Use `internalMutation`/`internalAction` for sensitive ops | Expose sensitive operations as public functions |
| Validate args with strict `v.*` validators                | Use `v.any()` for sensitive data                |
| Check ownership before update/delete                      | Trust that the caller owns the resource         |
| Store API keys in environment variables                   | Hardcode secrets in code                        |
| Log sensitive operations for audit trails                 | Skip audit logging                              |
| Rate limit user-facing endpoints                          | Allow unlimited requests                        |
| Use confirmation codes for destructive actions            | Allow single-click destructive operations       |
| Sanitize error messages for clients                       | Expose internal error details or stack traces   |

## References

- Convex Documentation: https://docs.convex.dev/
- Convex LLMs.txt: https://docs.convex.dev/llms.txt
- Authentication: https://docs.convex.dev/auth
- Functions Auth: https://docs.convex.dev/auth/functions-auth
- Production Security: https://docs.convex.dev/production
