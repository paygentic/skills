---
name: explore
description: >
  Use when a developer wants to understand Paygentic, evaluate its billing and metering
  capabilities, or learn how usage-based billing works with Paygentic. Use this skill
  whenever someone mentions Paygentic and asks conceptual questions like "how does
  metering work", "what is the billing model", "how do subscriptions work", "what can
  Paygentic do", or wants to explore before writing integration code. Also use when
  someone asks about consumption-based billing, usage events, entitlements, or
  billing metrics in a Paygentic context, even if they don't explicitly say "explore".
---

# Paygentic Explore

Help developers understand Paygentic's billing platform before they write integration code.

Answer from the information in this skill. Do not explore internal implementation details or codebase internals — merchants don't need to know about PubSub topics, ClickHouse tables, or Go services.

## First: Understand Where the Developer Is

Before explaining anything, ask:

> Where are you in your Paygentic journey?
> 1. **Just evaluating** — I want to understand what Paygentic does
> 2. **Have an account** — I have API keys but haven't started coding
> 3. **SDK installed** — I'm ready to write integration code
> 4. **Already integrated** — I need help with a specific operation

- Options 1-2: Continue with this skill
- Options 3-4: Switch to the `integrate` skill instead

If the developer's message already makes their stage obvious, skip the question and proceed.

## What Paygentic Does

Paygentic is a consumption-based billing platform for SaaS, AI, and API companies. It handles the full billing lifecycle: define what you charge for, track how much customers use, generate invoices, and collect payment.

Think of it as: **Stripe handles payments, Paygentic handles everything between "customer used your product" and "customer gets charged correctly."**

## Common Billing Patterns

**Credit-based / Quota billing** (most common for AI/API companies):
- Customer pays $49/mo and gets 1,000 API calls included
- Usage is tracked via meter events, balance is managed via entitlements and grants
- When credits run out: either block access (hard limit) or bill for overage (soft limit)
- Grants auto-reset each billing period with optional rollover

**Pure metered billing:**
- No included credits — customer pays per unit consumed (e.g., $0.01/API call)
- Usage tracked via meter events, invoiced at period end

**Hybrid:**
- Fixed subscription fee (billed upfront) + metered usage (billed in arrears)
- Example: $99/mo platform fee + $0.001/token

All three patterns use the same building blocks below.

## The Domain Model

Paygentic has two layers: things you set up once (catalog), and things your code handles at runtime.

### Catalog (configured in the Paygentic dashboard)

Set these up in the UI at [platform.paygentic.io](https://platform.paygentic.io) — no code needed:

- **Products** — what you sell (e.g., "API Platform", "Pro Tier")
- **Features** — capabilities within a product (e.g., "image-generation", "priority-support"). Can be boolean (on/off) or metered (has a balance).
- **Billable Metrics** — how you measure usage (e.g., "API calls", "tokens processed", "GB stored"). Defines aggregation type (SUM, COUNT, MAX) and unit.
- **Prices** — how much a metric or fee costs (flat rate, per-unit, tiered, graduated)
- **Plans** — bundles of prices into a purchasable package (e.g., "Pro Plan: $49/mo + $0.01/API call")

### Runtime (your code via SDK)

These operations happen in your application and require SDK integration:

- **Customers** — create and sync customers from your user model
- **Subscriptions** — subscribe customers to plans, manage lifecycle (create, terminate)
- **Meter Events** — send usage data via `events.ingest()` (fire-and-forget, CloudEvents format)
- **Entitlements** — check if a customer can access a feature and track their balance ("does user X have credits left?")
- **Grants** — allocate credits/quota to customers (e.g., "1000 API calls included per month"). Auto-issued from plans, with reset periods and optional rollover.
- **Invoices** — auto-generated from subscriptions; you can read them and add manual line items
- **Payments** — create one-off charges with checkout URLs

### Testing

- **Test Clocks** — simulate time passing to test billing cycles, renewals, and invoice generation without waiting

## Typical Integration Flow

Here's the order most merchants follow:

**1. Dashboard setup (no code):**
- Create your product, features, and billable metrics
- Define prices and bundle them into plans

**2. Backend integration (SDK code):**
- Sync customers from your user model → `customers.create()`
- Create subscriptions when customers sign up → `subscriptions.create()`
- Check entitlement balance before allowing operations → `entitlements.list()`
- Send usage events as customers use your product → `events.ingest()`
- (For credit-based: plan auto-issues grants each period, entitlements track remaining balance)

**3. Automated (Paygentic handles it):**
- Invoice generation at billing period end
- Payment collection via Stripe
- Revenue analytics and reporting

## API Versions

Paygentic is transitioning from v0 to v1 billing:

- **v1 billing** is the current path for new merchants
- Use `events.ingest()` for metering (v1) — NOT `usageEvents.*` (v0-only, will error on v1 plans)
- Use `entitlements.*` (not `entitlementsV0.*`) for feature access

## SDKs Available

| Language | Package | Install |
|----------|---------|---------|
| TypeScript | `@paygentic/sdk` | `npm install @paygentic/sdk` |
| Python | `paygentic-sdk` | `pip install paygentic-sdk` |

Both are generated from the OpenAPI spec via Speakeasy and stay in sync with the API.

## Learn More

- [Quickstart guide](https://docs.paygentic.io/getting-started/quickstart)
- [API reference](https://docs.paygentic.io/api-reference)
- [Billing concepts](https://docs.paygentic.io/platform/billing)
- [Metering guide](https://docs.paygentic.io/platform/metering)
- [Entitlements](https://docs.paygentic.io/platform/billing/entitlements)
- [Pricing models](https://docs.paygentic.io/platform/pricing)

## Ready to Code?

When the developer is ready to write integration code, point them to the `integrate` skill. It will detect their SDK language, ask about their setup state, and generate working code.
