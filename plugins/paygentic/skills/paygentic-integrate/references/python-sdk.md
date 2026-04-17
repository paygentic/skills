# Python SDK Reference (paygentic-sdk)

## Installation & Initialization

```bash
pip install paygentic-sdk
```

### Sync
```python
import os
from paygentic_sdk import Paygentic

paygentic = Paygentic(
    bearer_auth=os.getenv("PAYGENTIC_BEARER_AUTH", ""),
)
```

### Async
```python
import asyncio
from paygentic_sdk import Paygentic

async def main():
    async with Paygentic(
        bearer_auth=os.getenv("PAYGENTIC_BEARER_AUTH", ""),
    ) as paygentic:
        result = await paygentic.events.ingest_async(...)

asyncio.run(main())
```

Every method has a sync version (`method()`) and an async version (`method_async()`).

## Key Differences from TypeScript SDK

- **snake_case** parameter names: `merchant_id`, `customer_id`, `plan_id`, etc.
- **`type_`** with trailing underscore for `events.ingest()` (Python reserved word)
- **`from_`** with trailing underscore for `billable_metrics.meter()` (Python reserved word)
- **`invoices_v2`** instead of `invoicesV2`
- **`billable_metrics`** instead of `billableMetrics`
- **`test_clocks`** instead of `testClocks`

## Method Signatures

### events.ingest()
```python
paygentic.events.ingest(
    type_="api-call",            # note trailing underscore
    source="my-api-service",
    subject="customer_abc123",
    data={"tokens": 1500, "model": "gpt-4"},
    namespace=None,              # optional
    timestamp=None,              # optional
    idempotency_key=None,        # optional
)
# Returns: EventResponse. Always 202 — fire-and-forget.
```

### customers.create()
```python
paygentic.customers.create(
    merchant_id="merchant_xyz",
    consumer={
        "name": "Acme Corp",
        "email": "billing@acme.com",
        "address": {
            "line1": "123 Main St",
            "city": "San Francisco",
            "state": "CA",
            "postal_code": "94105",
            "country": "US",
        },
    },
    external_id="acme-123",      # optional
)
```

### subscriptions.create()
```python
from datetime import datetime

paygentic.subscriptions.create(
    name="Acme Pro Subscription",
    plan_id="plan_pro_monthly",
    started_at=datetime.now(),
    customer_id="customer_abc123",
    auto_charge=True,
)
```

### subscriptions.terminate()
```python
paygentic.subscriptions.terminate(
    id="sub_abc123",
    reason="Customer requested cancellation",
)
```

### subscriptions.generate_portal_link()
```python
portal = paygentic.subscriptions.generate_portal_link(
    id="sub_abc123",
    expires_in="PT1H",           # optional, ISO 8601 duration
)
# portal.url — time-limited portal URL
```

### entitlements.list()
```python
entitlements = paygentic.entitlements.list(
    customer_id="customer_abc123",
    feature_key="image-generation",
    limit=10,
    offset=0,
)
```

### entitlements.issue()
```python
paygentic.entitlements.issue(
    customer_id="customer_abc123",
    feature_id="feature_img_gen",
    template={"type": "metered", "limit": 1000},
)
```

### billable_metrics.meter()
```python
from datetime import datetime

usage = paygentic.billable_metrics.meter(
    id="metric_abc123",
    from_=datetime(2026, 4, 1),  # note trailing underscore
    to=datetime(2026, 4, 30),
    subject="customer_abc123",
    window_size="DAY",
)
```

### entitlements.grants.create()
```python
paygentic.entitlements.grants.create(
    entitlement_id="ent_abc123",
    amount=500,                      # units to grant
)
```

### entitlements.grants.purchase()
```python
purchase = paygentic.entitlements.grants.purchase(
    entitlement_id="ent_abc123",
    amount=1000,                     # units to purchase
)
# purchase — creates ad-hoc invoice with payment session
```

### entitlements.grants.list()
```python
grants = paygentic.entitlements.grants.list(
    entitlement_id="ent_abc123",
    limit=10,
    offset=0,
)
```

### entitlements.grants.void()
```python
paygentic.entitlements.grants.void(
    entitlement_id="ent_abc123",
    grant_id="grant_abc123",
)
```

### invoices_v2.list() / invoices_v2.get()
```python
invoices = paygentic.invoices_v2.list()

invoice = paygentic.invoices_v2.get(
    id="inv_abc123",
    expand="lineItems",
)
```

### invoices_v2.create_line_item()
```python
paygentic.invoices_v2.create_line_item(
    invoice_id="inv_abc123",
    description="Custom consulting fee",
    amount="250.00",
    currency="USD",
)
```

### payments.create()
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
# payment.checkout_url — redirect customer here
```

## Error Handling

```python
from paygentic_sdk.errors import PaygenticError

try:
    paygentic.customers.create(...)
except PaygenticError as e:
    print(f"Status {e.status_code}: {e.message}")
    # e.body — raw response body
    # e.raw_response — full httpx.Response
```

## Pagination

```python
page1 = paygentic.customers.list(limit=10, offset=0)
page2 = paygentic.customers.list(limit=10, offset=10)
```

## Context Managers

Always use context managers for proper resource cleanup:

```python
# Sync
with Paygentic(bearer_auth="...") as paygentic:
    paygentic.customers.list()

# Async
async with Paygentic(bearer_auth="...") as paygentic:
    await paygentic.customers.list_async()
```
