# TypeScript SDK Reference (@paygentic/sdk)

## Installation & Initialization

```bash
npm install @paygentic/sdk
```

```typescript
import { Paygentic } from "@paygentic/sdk";

const paygentic = new Paygentic({
  bearerAuth: process.env.PAYGENTIC_BEARER_AUTH,
});
```

All methods are async and return Promises.

## Method Signatures

### events.ingest()
```typescript
await paygentic.events.ingest({
  type: string,              // CloudEvents type — must match a billable metric event type
  source: string,            // URI identifying the event source
  subject: string,           // customer/entity ID
  data: Record<string, any>, // event payload with usage values
  namespace?: string,        // organization ID (optional, inferred from auth)
  timestamp?: Date,          // event time (optional, defaults to now)
  idempotencyKey?: string,   // deduplication key (optional)
});
// Returns: EventResponse. Always 202 — fire-and-forget.
```

### customers.create()
```typescript
await paygentic.customers.create({
  merchantId: string,            // required
  consumer?: {                   // new consumer details
    name: string,
    email: string,
    phone?: string,
    address: {
      line1: string,
      city: string,
      state: string,
      postalCode: string,
      country: string,           // ISO 3166-1 alpha-2
    },
  },
  consumerId?: string,           // OR reference existing consumer
  externalId?: string,           // your internal ID
  taxId?: string,                // tax registration ID
  taxRates?: Record<string, number>,
});
```

### subscriptions.create()
```typescript
await paygentic.subscriptions.create({
  name: string,                  // required
  planId: string,                // required — from dashboard
  startedAt: Date,               // required
  customerId?: string,           // existing customer
  customer?: { ... },            // OR create inline
  autoCharge?: boolean,          // default: false
  taxExempt?: boolean,           // default: false
  endingAt?: Date,               // fixed-term end date
  minimumAccountBalance?: string,
  testClockId?: string,          // for time-travel testing
  sessionExpiryMinutes?: number, // payment session expiry (default: 240)
});
```

### subscriptions.terminate()
```typescript
await paygentic.subscriptions.terminate({
  id: string,                    // subscription ID
  requestBody: {
    reason: string,              // cancellation explanation
  },
});
// Returns: Subscription
```

### subscriptions.generatePortalLink()
```typescript
await paygentic.subscriptions.generatePortalLink({
  id: string,                    // subscription ID
  requestBody?: {
    expiresIn?: string,          // ISO 8601 duration (default: PT10M, max: P7D)
  },
});
// Returns: { url: string } — time-limited portal URL
```

### entitlements.list()
```typescript
await paygentic.entitlements.list({
  customerId: string,            // required
  featureKey?: string,           // filter by feature key
  productId?: string,
  subscriptionId?: string,
  limit?: number,                // default: 10
  offset?: number,               // default: 0
  at?: Date,                     // evaluate at point in time
});
```

### entitlements.issue()
```typescript
await paygentic.entitlements.issue({
  customerId: string,
  featureId: string,
  template: {
    type: "metered" | "boolean",
    limit?: number,              // for metered: total allowed
  },
  activeFrom?: Date,
  activeTo?: Date,
  subscriptionId?: string,
  metadata?: Record<string, string>,
});
```

### billableMetrics.meter()
```typescript
await paygentic.billableMetrics.meter({
  id: string,                    // metric ID
  from: Date,                    // query window start
  to: Date,                      // query window end
  subject?: string,              // filter by customer
  windowSize?: "MINUTE" | "HOUR" | "DAY",
  groupBy?: string,              // comma-separated dimension keys
  filterGroupBy?: string,        // JSON-encoded dimension filter
});
```

### invoicesV2.list() / invoicesV2.get()
```typescript
await paygentic.invoicesV2.list({
  subscriptionId?: string,
  customerId?: string,
  status?: string,
});

await paygentic.invoicesV2.get({
  id: string,
  expand?: string,               // e.g., "lineItems"
  lineItemsLimit?: number,       // default: 100
  lineItemsPageToken?: string,
});
```

### payments.create()
```typescript
await paygentic.payments.create({
  amount: string,                // decimal string e.g. "10.50"
  currency: "USD" | "EUR" | "GBP" | "AUD",
  customerId?: string,
  merchantId?: string,
  idempotencyKey?: string,
  reference?: string,
  metadata?: Record<string, string>,
  lineItems?: Array<{ description: string, amount: string, quantity?: number }>,
  successRedirectUrl?: string,
  failureRedirectUrl?: string,
  savePaymentMethod?: boolean,   // default: false
  expiresIn?: string,            // ISO 8601 duration (default: P30D)
});
// Returns: Payment with checkoutUrl
```

## Error Handling

The SDK throws typed errors inheriting from `PaygenticError`:

```typescript
import { PaygenticError } from "@paygentic/sdk/models/errors/paygenticerror";

try {
  await paygentic.customers.create({ ... });
} catch (error) {
  if (error instanceof PaygenticError) {
    console.error(error.statusCode, error.message, error.body);
  }
}
```

## Pagination

List methods support `limit` and `offset`:

```typescript
const page1 = await paygentic.customers.list({ limit: 10, offset: 0 });
const page2 = await paygentic.customers.list({ limit: 10, offset: 10 });
```
