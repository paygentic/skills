---
name: paygentic-integrate
description: >
  Use when a developer wants to integrate Paygentic into their application using
  the SDK. Use this skill whenever someone wants to send usage events, create
  customers, manage subscriptions, check entitlements, retrieve invoices, or set up
  payments with Paygentic. Also use when someone says "set up billing", "add
  metering", "wire up Paygentic", "integrate Paygentic", or has the @paygentic/sdk
  or paygentic-sdk package installed. Even if they just say "I want to bill my
  users" in a project that has Paygentic as a dependency, use this skill.
---

# Paygentic Integrate

Write SDK integration code for merchant developers. Detect their language, understand their setup state, and generate working code.

## First: Understand the Developer's Context

Before writing any code, determine:

1. **Setup state** — Do they have an API key and SDK installed?
   - No API key → direct them to [platform.paygentic.io](https://platform.paygentic.io) to create one
   - No SDK → show install command (see SDK Setup below)
   - Ready → proceed to their requested operation

2. **Language** — Check the project for signals:
   - `package.json` with `@paygentic/sdk` → TypeScript
   - `requirements.txt` / `pyproject.toml` with `paygentic-sdk` → Python
   - If unclear, ask which SDK they want to use

3. **What they want to do** — metering, customers, subscriptions, entitlements, invoices, or payments

If the developer's message already makes context obvious, skip questions and proceed.

After determining the language, read the appropriate reference file for SDK-specific patterns:
- TypeScript: `references/typescript-sdk.md`
- Python: `references/python-sdk.md`

## SDK Setup

### TypeScript
```bash
npm install @paygentic/sdk
```
```typescript
import { Paygentic } from "@paygentic/sdk";

const paygentic = new Paygentic({
  bearerAuth: process.env.PAYGENTIC_BEARER_AUTH,
});
```

### Python
```bash
pip install paygentic-sdk
```
```python
import os
from paygentic_sdk import Paygentic

paygentic = Paygentic(
    bearer_auth=os.getenv("PAYGENTIC_BEARER_AUTH", ""),
)
```

Python supports both sync and async. Every method has an `_async` variant (e.g., `customers.create_async()`). Use `async with Paygentic(...) as paygentic:` for async context management.

## Usage Metering (P0)

The most common integration point. Merchants send meter events as customers use their product.

**Endpoint:** `events.ingest()` — fire-and-forget. Always returns 202 Accepted. Uses CloudEvents format.

**IMPORTANT:** Do NOT use `usageEvents.create()` or `usageEvents.batchCreate()` — those are v0-only and will error on v1 billing plans.

### TypeScript
```typescript
await paygentic.events.ingest({
  type: "api-call",                    // matches a billable metric's event type
  source: "my-api-service",            // identifies the sending service
  subject: "customer_abc123",          // the customer/entity being metered
  data: {
    tokens: 1500,                      // usage amount — key must match metric config
    model: "gpt-4",                    // optional dimensions for grouping/filtering
  },
});
// Always returns 202 — no confirmation, fire-and-forget
```

### Python
```python
paygentic.events.ingest(
    type_="api-call",                  # note: type_ with underscore (Python reserved word)
    source="my-api-service",
    subject="customer_abc123",
    data={
        "tokens": 1500,
        "model": "gpt-4",
    },
)
```

### Querying Usage

Use `billableMetrics.meter()` to read aggregated usage data:

```typescript
const usage = await paygentic.billableMetrics.meter({
  id: "metric_abc123",                // billable metric ID
  from: new Date("2026-04-01"),       // start of query window
  to: new Date("2026-04-30"),         // end of query window
  subject: "customer_abc123",         // optional: filter by customer
  windowSize: "DAY",                  // optional: bucket by DAY, HOUR, MINUTE
});
```

## Customers (P0)

Sync customers from the merchant's own user model into Paygentic.

### TypeScript
```typescript
const customer = await paygentic.customers.create({
  merchantId: "merchant_xyz",
  consumer: {
    name: "Acme Corp",
    email: "billing@acme.com",
    address: {
      line1: "123 Main St",
      city: "San Francisco",
      state: "CA",
      postalCode: "94105",
      country: "US",
    },
  },
  externalId: "acme-123",             // your internal ID for cross-referencing
});
```

Other operations: `customers.list()`, `customers.get()`, `customers.update()`, `customers.delete()`.

## Subscriptions (P0)

Subscribe customers to plans. Plans are created in the dashboard.

### TypeScript
```typescript
const subscription = await paygentic.subscriptions.create({
  name: "Acme Pro Subscription",
  planId: "plan_pro_monthly",          // from dashboard
  customerId: customer.id,
  startedAt: new Date(),
  autoCharge: true,                    // charge automatically at period end
});
```

### Terminate a subscription
```typescript
await paygentic.subscriptions.terminate({
  id: subscription.id,
  requestBody: {
    reason: "Customer requested cancellation",
  },
});
```

### Customer portal link
Generate a self-service portal URL for customers to manage their subscription:
```typescript
const portal = await paygentic.subscriptions.generatePortalLink({
  id: subscription.id,
});
// portal.url — time-limited URL (default 10 minutes, max 7 days)
```

Other operations: `subscriptions.list()`, `subscriptions.get()`.

## Credit-Based Billing (P0 — Most Common Pattern)

The most common integration pattern: customer gets N credits/month included in their plan, usage is tracked, and access is gated when credits run out (or overage is billed).

This combines metering + entitlements + grants into one flow.

### How it works

1. **Dashboard**: Create a metered feature (e.g., `api-calls`) linked to a billable metric. Attach an entitlement template to the price with grant amount and reset period.
2. **SDK**: On each request, check the entitlement balance → if credits remain, allow and ingest the event → Paygentic decrements the balance automatically.

### Check balance before allowing a request
```typescript
const entitlements = await paygentic.entitlements.list({
  customerId: "customer_abc123",
  featureKey: "api-calls",
});

const entitlement = entitlements.data[0];
if (!entitlement || entitlement.balance <= 0) {
  // No credits left — block or trigger overage
  return res.status(429).json({ error: "API call quota exceeded" });
}

// Credits available — proceed with the request, then track usage
await paygentic.events.ingest({
  type: "api.call",
  source: "my-api-service",
  subject: "customer_abc123",
  data: { endpoint: req.path },
});
```

### Hard limits vs soft limits

- **Hard limit** (`isSoftLimit: false`): Access blocked when balance hits zero. Use for free-tier caps.
- **Soft limit** (`isSoftLimit: true`): Access continues at zero balance, overage is tracked and billed. Use for "1000 included, then $0.01 each."

### Add credits on-demand (grants)
```typescript
// Top up a customer with additional credits
const grant = await paygentic.entitlements.grants.create({
  entitlementId: entitlement.id,
  amount: 500,                         // additional units
});

// Or let the customer purchase credits (creates invoice + payment session)
const purchase = await paygentic.entitlements.grants.purchase({
  entitlementId: entitlement.id,
  amount: 1000,
});
// purchase — creates ad-hoc invoice with payment session
```

### List and void grants
```typescript
const grants = await paygentic.entitlements.grants.list({
  entitlementId: entitlement.id,
});

// Void a grant (removes from balance)
await paygentic.entitlements.grants.void({
  entitlementId: entitlement.id,
  grantId: grant.id,
});
```

## Entitlements (P0)

Beyond credit-based billing, entitlements also support boolean feature gating (on/off access).

### Check feature access
```typescript
const entitlements = await paygentic.entitlements.list({
  customerId: "customer_abc123",
  featureKey: "priority-support",      // boolean feature
});

const hasAccess = entitlements.data.length > 0;
```

### Issue an entitlement manually
```typescript
await paygentic.entitlements.issue({
  customerId: "customer_abc123",
  featureId: "feature_priority_support",
  template: {
    type: "boolean",
  },
});
```

## Invoices (P1)

Invoices are auto-generated from subscriptions. You can read them and add manual line items.

### TypeScript
```typescript
// List invoices for a subscription
const invoices = await paygentic.invoicesV2.list({
  subscriptionId: "sub_abc123",
});

// Get a specific invoice with line items
const invoice = await paygentic.invoicesV2.get({
  id: "inv_abc123",
  expand: "lineItems",
});

// Add a manual line item (ad-hoc charge or credit)
await paygentic.invoicesV2.createLineItem({
  invoiceId: "inv_abc123",
  description: "Custom consulting fee",
  amount: "250.00",
  currency: "USD",
});
```

### Python
```python
# List invoices for a subscription
invoices = paygentic.invoices_v2.list(
    subscription_id="sub_abc123",
)

# Get a specific invoice with line items
invoice = paygentic.invoices_v2.get(
    id="inv_abc123",
    expand="lineItems",
)

# Add a manual line item (ad-hoc charge or credit)
paygentic.invoices_v2.create_line_item(
    invoice_id="inv_abc123",
    description="Custom consulting fee",
    amount="250.00",
    currency="USD",
)
```

## Payments (P1)

Create one-off payment charges. Returns a checkout URL the customer visits to pay.

### TypeScript
```typescript
const payment = await paygentic.payments.create({
  amount: "99.00",
  currency: "USD",
  customerId: "customer_abc123",
  reference: "onboarding-fee",
  successRedirectUrl: "https://myapp.com/payment/success",
  failureRedirectUrl: "https://myapp.com/payment/failed",
  lineItems: [
    { description: "Onboarding fee", amount: "99.00" },
  ],
});
// payment.checkoutUrl — redirect customer here to complete payment
```

### Python
```python
payment = paygentic.payments.create(
    amount="99.00",
    currency="USD",
    customer_id="customer_abc123",
    reference="onboarding-fee",
    success_redirect_url="https://myapp.com/payment/success",
    failure_redirect_url="https://myapp.com/payment/failed",
    line_items=[
        {"description": "Onboarding fee", "amount": "99.00"},
    ],
)
# payment.checkout_url — redirect customer here to complete payment
```

## Additional Domains (P2 — Available, Expand Later)

These are available in the SDK but covered lightly here. Ask if the developer needs help with any:

- **Sources** — connect external data sources for automated usage ingestion with approval workflow (`sources.create()`, `sources.events.approve()`, `sources.rules.create()`)
- **Disputes** — create disputes for contested usage events (`disputes.create()`, `disputes.list()`)
- **Costs** — track infrastructure/operational costs against usage (`costs.createCost()`, `costs.getCostSummary()`)
- **Test Clocks** — simulate time for testing billing cycles (`testClocks.create()`, `testClocks.advance()`)

## Common Mistakes

1. **Using v0 usage endpoints with v1 plans** — `usageEvents.create()` and `usageEvents.batchCreate()` will return an error for v1 billing subscriptions. Always use `events.ingest()`.

2. **Expecting confirmation from ingest** — `events.ingest()` always returns 202 Accepted with no validation. If the event type doesn't match a metric, it's silently ignored. Verify metric configuration in the dashboard.

3. **Missing idempotency** — For critical metering, pass `idempotencyKey` to `events.ingest()` to prevent duplicate charges on retries.

4. **Not using the portal link** — Instead of building custom subscription management UI, use `subscriptions.generatePortalLink()` to give customers self-service access.

## Learn More

- [API reference](https://docs.paygentic.io/api-reference)
- [Metering guide](https://docs.paygentic.io/platform/metering)
- [Billing concepts](https://docs.paygentic.io/platform/billing)
- [Entitlements](https://docs.paygentic.io/platform/billing/entitlements)
