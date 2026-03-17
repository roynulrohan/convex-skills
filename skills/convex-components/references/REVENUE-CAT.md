# RevenueCat Component

Handle RevenueCat webhooks and store subscription state in Convex with real-time reactivity for subscription access control.

**Note:** This is a community component (`convex-revenuecat`), not an official `@convex-dev/*` package.

## Installation

```bash
npm install convex-revenuecat
```

```typescript
// convex/convex.config.ts
import revenuecat from "convex-revenuecat/convex.config";
app.use(revenuecat);
```

## Setup

### 1. Generate Webhook Secret

```bash
openssl rand -base64 32
npx convex env set REVENUECAT_WEBHOOK_AUTH "your-generated-secret"
```

### 2. Mount Webhook Handler

```typescript
// convex/http.ts
import { httpRouter } from "convex/server";
import { RevenueCat } from "convex-revenuecat";
import { components } from "./_generated/api";

const http = httpRouter();
const revenuecat = new RevenueCat(components.revenuecat, {
  REVENUECAT_WEBHOOK_AUTH: process.env.REVENUECAT_WEBHOOK_AUTH,
});

http.route({
  path: "/webhooks/revenuecat",
  method: "POST",
  handler: revenuecat.httpHandler(),
});

export default http;
```

### 3. Configure RevenueCat Dashboard

1. Go to RevenueCat Dashboard → Project Settings → Webhooks
2. Add endpoint: `https://<your-deployment>.convex.site/webhooks/revenuecat`
3. Set Authorization header to `Bearer <your-generated-secret>`

## Constructor

```typescript
const revenuecat = new RevenueCat(components.revenuecat, {
    REVENUECAT_WEBHOOK_AUTH?: string, // If omitted, webhook auth is skipped
});
```

## Query Methods

All methods take `ctx` as first argument and work with query/action contexts.

### Entitlements

```typescript
// Check if user has specific entitlement
const hasPremium = await revenuecat.hasEntitlement(ctx, {
  appUserId: "user_123",
  entitlementId: "premium",
});

// Get all active entitlements
const active = await revenuecat.getActiveEntitlements(ctx, {
  appUserId: "user_123",
});

// Get all entitlements (including expired)
const all = await revenuecat.getAllEntitlements(ctx, {
  appUserId: "user_123",
});
```

### Subscriptions

```typescript
// Active subscriptions only
const activeSubs = await revenuecat.getActiveSubscriptions(ctx, {
  appUserId: "user_123",
});

// All subscriptions (including expired/canceled)
const allSubs = await revenuecat.getAllSubscriptions(ctx, {
  appUserId: "user_123",
});

// Subscriptions in grace period
const gracePeriod = await revenuecat.getSubscriptionsInGracePeriod(ctx, {
  appUserId: "user_123",
});

// Check specific subscription's grace period status
const status = await revenuecat.isInGracePeriod(ctx, {
  originalTransactionId: "txn_123",
});
// Returns: { inGracePeriod: boolean, gracePeriodExpiresAt?: number, billingIssueDetectedAt?: number }
```

### Customer

```typescript
const customer = await revenuecat.getCustomer(ctx, {
  appUserId: "user_123",
});
// Returns: Customer | null
```

### Experiments (A/B Tests)

```typescript
const experiment = await revenuecat.getExperiment(ctx, {
  appUserId: "user_123",
  experimentId: "paywall_v2",
});

const allExperiments = await revenuecat.getExperiments(ctx, {
  appUserId: "user_123",
});
```

### Transfers

```typescript
const transfer = await revenuecat.getTransfer(ctx, { eventId: "evt_123" });

const transfers = await revenuecat.getTransfers(ctx, { limit: 50 }); // default: 100
```

### Invoices

```typescript
const invoice = await revenuecat.getInvoice(ctx, { invoiceId: "inv_123" });

const invoices = await revenuecat.getInvoices(ctx, { appUserId: "user_123" });
```

### Virtual Currency

```typescript
const balance = await revenuecat.getVirtualCurrencyBalance(ctx, {
  appUserId: "user_123",
  currencyCode: "COINS",
});

const allBalances = await revenuecat.getVirtualCurrencyBalances(ctx, {
  appUserId: "user_123",
});

const transactions = await revenuecat.getVirtualCurrencyTransactions(ctx, {
  appUserId: "user_123",
  currencyCode: "COINS", // optional filter
});
```

## Full API Reference

| Method                           | Args                       | Returns                          |
| -------------------------------- | -------------------------- | -------------------------------- |
| `hasEntitlement`                 | `appUserId, entitlementId` | `boolean`                        |
| `getActiveEntitlements`          | `appUserId`                | `Entitlement[]`                  |
| `getAllEntitlements`             | `appUserId`                | `Entitlement[]`                  |
| `getActiveSubscriptions`         | `appUserId`                | `Subscription[]`                 |
| `getAllSubscriptions`            | `appUserId`                | `Subscription[]`                 |
| `getSubscriptionsInGracePeriod`  | `appUserId`                | `Subscription[]`                 |
| `isInGracePeriod`                | `originalTransactionId`    | `GracePeriodStatus`              |
| `getCustomer`                    | `appUserId`                | `Customer \| null`               |
| `getExperiment`                  | `appUserId, experimentId`  | `Experiment \| null`             |
| `getExperiments`                 | `appUserId`                | `Experiment[]`                   |
| `getTransfer`                    | `eventId`                  | `Transfer \| null`               |
| `getTransfers`                   | `limit?`                   | `Transfer[]`                     |
| `getInvoice`                     | `invoiceId`                | `Invoice \| null`                |
| `getInvoices`                    | `appUserId`                | `Invoice[]`                      |
| `getVirtualCurrencyBalance`      | `appUserId, currencyCode`  | `VirtualCurrencyBalance \| null` |
| `getVirtualCurrencyBalances`     | `appUserId`                | `VirtualCurrencyBalance[]`       |
| `getVirtualCurrencyTransactions` | `appUserId, currencyCode?` | `VirtualCurrencyTransaction[]`   |

## Common Patterns

### Check Premium Access

```typescript
import { query } from "./_generated/server";
import { components } from "./_generated/api";
import { RevenueCat } from "convex-revenuecat";
import { v } from "convex/values";

const revenuecat = new RevenueCat(components.revenuecat);

export const checkPremium = query({
  args: { appUserId: v.string() },
  returns: v.boolean(),
  handler: async (ctx, args) => {
    return await revenuecat.hasEntitlement(ctx, {
      appUserId: args.appUserId,
      entitlementId: "premium",
    });
  },
});
```

### Gate Feature Access

```typescript
export const getPremiumContent = query({
  args: { appUserId: v.string() },
  handler: async (ctx, { appUserId }) => {
    const hasPremium = await revenuecat.hasEntitlement(ctx, {
      appUserId,
      entitlementId: "premium",
    });

    if (!hasPremium) {
      return { locked: true, content: null };
    }

    const content = await ctx.db.query("premiumContent").collect();
    return { locked: false, content };
  },
});
```

## Webhook Event Types

All 18 RevenueCat event types are handled:

| Event                          | Behavior                                               |
| ------------------------------ | ------------------------------------------------------ |
| `INITIAL_PURCHASE`             | Creates subscription + entitlements                    |
| `RENEWAL`                      | Updates subscription period                            |
| `CANCELLATION`                 | Marks cancel but **preserves access until expiration** |
| `EXPIRATION`                   | Revokes entitlements                                   |
| `BILLING_ISSUE`                | Starts grace period, **keeps access**                  |
| `SUBSCRIPTION_PAUSED`          | **Does not revoke** entitlements                       |
| `SUBSCRIPTION_EXTENDED`        | Extends subscription period                            |
| `TRANSFER`                     | Moves entitlements between users                       |
| `UNCANCELLATION`               | Removes cancellation                                   |
| `PRODUCT_CHANGE`               | Updates subscription product                           |
| `NON_RENEWING_PURCHASE`        | Creates one-time entitlement                           |
| `TEMPORARY_ENTITLEMENT_GRANT`  | Creates temporary access                               |
| `REFUND`                       | Revokes entitlements                                   |
| `REFUND_REVERSED`              | Restores entitlements                                  |
| `TEST`                         | Logged, no data changes                                |
| `INVOICE_ISSUANCE`             | Stores invoice record                                  |
| `VIRTUAL_CURRENCY_TRANSACTION` | Updates currency balances                              |
| `EXPERIMENT_ENROLLMENT`        | Stores A/B test data                                   |

All events are processed **idempotently** (deduped by `eventId`).

## Exported Types

```typescript
import type {
  Store, // "APP_STORE" | "PLAY_STORE" | "STRIPE" | "AMAZON" | ...
  Environment, // "SANDBOX" | "PRODUCTION"
  PeriodType, // "TRIAL" | "INTRO" | "NORMAL" | "PROMOTIONAL" | "PREPAID"
  OwnershipType, // "PURCHASED" | "FAMILY_SHARED"
  Entitlement,
  Subscription,
  Customer,
  Experiment,
  Transfer,
  Invoice,
  VirtualCurrencyBalance,
  VirtualCurrencyTransaction,
  GracePeriodStatus,
} from "convex-revenuecat";
```

**Store values:** `"AMAZON"` | `"APP_STORE"` | `"MAC_APP_STORE"` | `"PADDLE"` | `"PLAY_STORE"` | `"PROMOTIONAL"` | `"RC_BILLING"` | `"ROKU"` | `"STRIPE"` | `"TEST_STORE"`

## Database Tables

The component manages these tables automatically:

- **customers** — user profiles with aliases and attributes
- **subscriptions** — full subscription state with store, period, renewal status
- **entitlements** — active/inactive entitlement tracking
- **webhookEvents** — event log with status (`processed` | `failed` | `ignored`)
- **experiments** — A/B test enrollment data
- **transfers** — subscription transfer records
- **invoices** — invoice records
- **virtualCurrencyBalances** — currency balance tracking
- **virtualCurrencyTransactions** — currency transaction history
- **rateLimits** — built-in rate limiting (100 req/min per app)

## Testing

```typescript
import { convexTest } from "convex-test";
import revenuecatTest from "convex-revenuecat/test";

const t = convexTest();
revenuecatTest.register(t); // Registers as "revenuecat"
// Or: revenuecatTest.register(t, "customName");
```

## Troubleshooting

**Webhooks not arriving:**

- Verify endpoint URL: `https://<deployment>.convex.site/webhooks/revenuecat`
- Check Authorization header uses `Bearer <secret>` format
- Verify `REVENUECAT_WEBHOOK_AUTH` env var matches

**User shows no entitlements:**

- Check `appUserId` matches what RevenueCat sends (may differ from your internal user ID)
- Verify webhook events are being processed: check `webhookEvents` table in dashboard

**Rate limiting (429 errors):**

- Built-in rate limit is 100 requests/minute per app
- RevenueCat will retry webhooks automatically
