# Go SDK Reference (github.com/paygentic/sdk-go)

Targets `v0.2.1`. Speakeasy-generated; types live in `models/components`, request structs in `models/operations`, errors in `models/errors`.

## Installation & Initialization

```bash
go get github.com/paygentic/sdk-go@v0.2.1
```

```go
import (
    "os"

    paygentic "github.com/paygentic/sdk-go"
)

// Sandbox (default for initial integration)
client := paygentic.New(
    paygentic.WithServerURL("https://api.sandbox.paygentic.io"),
    paygentic.WithSecurity(os.Getenv("PAYGENTIC_BEARER_AUTH")),
)

// Production — switch when ready to go live:
// client := paygentic.New(
//     paygentic.WithServerURL("https://api.paygentic.io"),
//     paygentic.WithSecurity(os.Getenv("PAYGENTIC_BEARER_AUTH")),
// )
```

`ServerList` indexes `0` (production) and `1` (sandbox), so `paygentic.WithServerIndex(1)` is equivalent to the sandbox `WithServerURL` above. Do **not** include `/v0` or `/v1` in the URL — the SDK appends the version per endpoint.

Every method takes a `context.Context` as the first argument. Use it to wire request cancellation and deadlines:

```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
res, err := client.Customers.Get(ctx, "cus_abc123")
```

## Key Differences from TypeScript / Python SDKs

- **Pointer helpers** for optional scalars: `paygentic.String(s)`, `paygentic.Bool(b)`, `paygentic.Int(i)`, `paygentic.Int64(i)`, `paygentic.Float32(f)`, `paygentic.Float64(f)`, plus the generic `paygentic.Pointer[T](v)`.
- **Time helper** for ISO-8601 string → `time.Time`: `types.MustTimeFromString("2026-04-01T00:00:00Z")` from `github.com/paygentic/sdk-go/types`.
- **Typed enums** instead of stringly-typed parameters — e.g. `operations.CurrencyUsd`, `operations.GetBillableMetricMeterWindowSizeDay`.
- **Positional `id` arguments** for resource-scoped methods (`Subscriptions.Terminate`, `Subscriptions.GeneratePortalLink`, `Subscriptions.Get`, `InvoicesV2.Get`, `Grants.*`, `Customers.Get/Update/Delete`). The TS/Python SDKs bundle `id` inside the request object; Go separates path params from the body.
- **`OptionalNullable[T]`** wrapper for nullable optional fields (e.g. `ExpiresAt`, `Description`). Imported from `github.com/paygentic/sdk-go/optionalnullable`. Construct with `optionalnullable.NewOptionalNullable(v)`; pass `nil` to omit.
- **Discriminated union responses** for some endpoints (e.g. `Customers.Create` returns `*operations.CreateCustomerResponse` with `.Type` field — switch on it before reading `Body1`/`Body2`).
- **Errors via `errors.As`** rather than `instanceof` / `except` — see [Error Handling](#error-handling) below.

## Imports used in the examples

```go
import (
    "context"
    "fmt"

    paygentic "github.com/paygentic/sdk-go"
    "github.com/paygentic/sdk-go/models/components"
    "github.com/paygentic/sdk-go/models/operations"
    paygenticErrors "github.com/paygentic/sdk-go/models/errors"
    "github.com/paygentic/sdk-go/optionalnullable"
    "github.com/paygentic/sdk-go/types"
)
```

## Method Signatures

### Events.Ingest

```go
res, err := client.Events.Ingest(ctx, operations.IngestEventRequest{
    Type:    "api-call",                         // matches a billable metric event type
    Source:  "my-api-service",
    Subject: "customer_abc123",
    Data: map[string]any{
        "tokens": 1500,
        "model":  "gpt-4",
    },
    // Optional:
    // Namespace:      paygentic.String("org_abc123"),
    // Timestamp:      paygentic.Pointer(time.Now()),
    // IdempotencyKey: paygentic.String("req_xyz"),
})
// Returns *components.EventResponse. Always 202 — fire-and-forget.
```

### Customers.Create

```go
res, err := client.Customers.Create(ctx, operations.CreateCustomerRequest{
    MerchantID: "merchant_xyz",
    Consumer: &operations.Consumer{
        Name:  "Acme Corp",
        Email: "billing@acme.com",
        Address: components.Address{
            Line1:   paygentic.String("123 Main St"),
            City:    paygentic.String("San Francisco"),
            State:   paygentic.String("CA"),
            ZipCode: paygentic.String("94105"),
            Country: paygentic.String("US"), // ISO 3166-1 alpha-2
        },
    },
    ExternalID: paygentic.String("acme-123"),
})
// res is *operations.CreateCustomerResponse — a discriminated union. Switch on Type:
switch res.Type {
case operations.CreateCustomerResponseTypeCreateCustomerResponseBody2:
    fmt.Println("created:", res.CreateCustomerResponseBody2.CustomerID)
case operations.CreateCustomerResponseTypeCreateCustomerResponseBody1:
    fmt.Println("already exists:", res.CreateCustomerResponseBody1.CustomerID)
case operations.CreateCustomerResponseTypeUnknown:
    // unrecognised payload shape — inspect res.UnknownRaw
}
```

Use `ConsumerID *string` instead of `Consumer` to attach an existing consumer.

### Subscriptions.Create

```go
sub, err := client.Subscriptions.Create(ctx, operations.CreateSubscriptionRequest{
    Name:       "Acme Pro Subscription",
    PlanID:     "plan_pro_monthly",
    CustomerID: paygentic.String("cus_abc123"),
    StartedAt:  types.MustTimeFromString("2026-04-17T00:00:00Z"),
    AutoCharge: paygentic.Bool(true),
})
// sub is *components.Subscription
```

### Subscriptions.Terminate

```go
sub, err := client.Subscriptions.Terminate(ctx, "sub_abc123", operations.TerminateSubscriptionRequestBody{
    Reason: "Customer requested cancellation",
})
// id is positional; the body holds Reason.
```

### Subscriptions.GeneratePortalLink

```go
portal, err := client.Subscriptions.GeneratePortalLink(ctx, "sub_abc123", &operations.GeneratePortalLinkRequestBody{
    ExpiresIn: paygentic.String("PT1H"), // ISO 8601 duration; default PT10M, max P7D
})
// portal.URL — time-limited URL
```

Pass `nil` for the body to use defaults.

### Entitlements.List

```go
res, err := client.Entitlements.List(ctx, operations.ListEntitlementsRequest{
    CustomerID: "cus_abc123",
    FeatureKey: paygentic.String("api-calls"),
    Limit:      paygentic.Int64(10),
    Offset:     paygentic.Int64(0),
})
```

### Entitlements.Issue

```go
ent, err := client.Entitlements.Issue(ctx, components.IssueEntitlementRequest{
    CustomerID: "cus_abc123",
    FeatureID:  "feature_priority_support",
    Template: components.CreateEntitlementTemplateBoolean(components.EntitlementTemplateBoolean{}),
    // Or: components.CreateEntitlementTemplateStatic(...) / CreateEntitlementTemplateMetered(...)
})
```

> Note: this method takes the request from `components`, not `operations`.

### BillableMetrics.Meter

```go
usage, err := client.BillableMetrics.Meter(ctx, operations.GetBillableMetricMeterRequest{
    ID:         "metric_abc123",
    From:       types.MustTimeFromString("2026-04-01T00:00:00Z"),
    To:         types.MustTimeFromString("2026-04-30T00:00:00Z"),
    Subject:    paygentic.String("cus_abc123"),
    WindowSize: operations.GetBillableMetricMeterWindowSizeDay.ToPointer(), // MINUTE | HOUR | DAY | MONTH
})
```

### Entitlements.Grants.Create

```go
grant, err := client.Entitlements.Grants.Create(ctx, "ent_abc123", components.CreateGrantRequest{
    Amount:         500,                       // float64
    IdempotencyKey: "grant_top_up_2026_04",    // required, unique per entitlement
    // EffectiveAt:   paygentic.Pointer(time.Now()),
    // ExpiresAt:     optionalnullable.NewOptionalNullable(types.MustTimeFromString("2027-01-01T00:00:00Z")),
})
// Note: no Metadata field on CreateGrantRequest in the Go SDK.
```

### Entitlements.Grants.Purchase

```go
purchase, err := client.Entitlements.Grants.Purchase(ctx, "ent_abc123", components.PurchaseGrantRequest{
    Amount:         1000,             // float64 — credits to grant on payment
    Price:          "5.00",           // decimal string, USD ≥ 0.50
    IdempotencyKey: "purchase_xyz",
    // Optional:
    // SuccessURL:        paygentic.String("https://myapp.com/success"),
    // CancelURL:         paygentic.String("https://myapp.com/cancel"),
    // PaymentExpiresAt:  paygentic.Pointer(time.Now().Add(24 * time.Hour)),
})
// purchase is *components.PurchaseGrantResponse — ad-hoc invoice + payment session.
```

### Entitlements.Grants.List

```go
grants, err := client.Entitlements.Grants.List(
    ctx,
    "ent_abc123",
    paygentic.Int64(10),  // limit
    paygentic.Int64(0),   // offset
    paygentic.Bool(false), // includeVoided
)
```

### Entitlements.Grants.Void

```go
voided, err := client.Entitlements.Grants.Void(ctx, "ent_abc123", "grant_abc123")
// voided is *components.Grant
```

### InvoicesV2.List

```go
list, err := client.InvoicesV2.List(ctx, &operations.ListInvoicesRequest{
    SubscriptionID: paygentic.String("sub_abc123"),
    Limit:          paygentic.Int64(10),
})
```

### InvoicesV2.Get

```go
invoice, err := client.InvoicesV2.Get(
    ctx,
    "inv_abc123",
    paygentic.String("lineItems"), // expand
    paygentic.Int64(100),          // lineItemsLimit
    nil,                            // lineItemsPageToken
)
```

### InvoicesV2.CreateLineItem

```go
item, err := client.InvoicesV2.CreateLineItem(
    ctx,
    components.CreateManualLineItemRequest{
        InvoiceID:   paygentic.String("inv_abc123"),
        DisplayName: "Custom consulting fee",
        Currency:    "USD",
        Quantity:    1,
        UnitPrice:   250.00,
        Description: optionalnullable.NewOptionalNullable("One-off consulting engagement"),
    },
    paygentic.String("line_item_2026_04"), // idempotencyKey (optional)
)
```

The line item must reference exactly one of `SubscriptionID` or `InvoiceID`. Unlike the TypeScript SDK's single `amount` string, Go uses `Quantity * UnitPrice` (both `float64`).

### Payments.Create

```go
payment, err := client.Payments.Create(ctx, operations.CreatePaymentRequest{
    Amount:             "99.00",
    Currency:           operations.CurrencyUsd, // typed enum
    CustomerID:         paygentic.String("cus_abc123"),
    Reference:          paygentic.String("onboarding-fee"),
    SuccessRedirectURL: paygentic.String("https://myapp.com/payment/success"),
    FailureRedirectURL: paygentic.String("https://myapp.com/payment/failed"),
    LineItems: []operations.LineItem{
        {Description: "Onboarding fee", Amount: "99.00"},
    },
})
// payment.CheckoutURL — redirect customer here to complete payment.
```

## Error Handling

The SDK returns concrete error types from `github.com/paygentic/sdk-go/models/errors`. Use `errors.As` to discriminate:

```go
import (
    "errors"

    paygenticErrors "github.com/paygentic/sdk-go/models/errors"
)

_, err := client.Customers.Create(ctx, req)
if err != nil {
    var badReq *paygenticErrors.BadRequest
    var validation *paygenticErrors.ValidationError
    var defaultErr *paygenticErrors.PaygenticDefaultError
    switch {
    case errors.As(err, &badReq):
        // 400 — discriminated union of Error / ValidationError
        fmt.Println("bad request:", badReq.Error())
    case errors.As(err, &validation):
        fmt.Println("validation:", validation.Error())
    case errors.As(err, &defaultErr):
        // Fallback for any non-2xx without a typed schema
        fmt.Println("status", defaultErr.StatusCode, "body", defaultErr.Body)
    default:
        fmt.Println("transport error:", err)
    }
}
```

`paygenticErrors.Error` is the base shape (`Message`, `Code`, `Details`) wrapped by `BadRequest`.

## Pagination

List methods take `limit *int64` and `offset *int64` — wrap with `paygentic.Int64`:

```go
page1, _ := client.Customers.List(ctx, operations.ListCustomersRequest{
    Limit:  paygentic.Int64(10),
    Offset: paygentic.Int64(0),
})
page2, _ := client.Customers.List(ctx, operations.ListCustomersRequest{
    Limit:  paygentic.Int64(10),
    Offset: paygentic.Int64(10),
})
```

## Pointer Helpers & Time Parsing

The SDK uses pointers for optional scalars to distinguish "unset" from the zero value. Build them inline:

```go
paygentic.String("foo")   // *string
paygentic.Int64(10)       // *int64
paygentic.Bool(true)      // *bool
paygentic.Float64(99.00)  // *float64
paygentic.Pointer(myEnum) // generic *T
```

For `time.Time` from string literals:

```go
import "github.com/paygentic/sdk-go/types"

types.MustTimeFromString("2026-04-01T00:00:00Z")     // time.Time (panics on bad input)
types.MustNewTimeFromString("2026-04-01T00:00:00Z")  // *time.Time
```

For `OptionalNullable[T]` fields (nullable + optional, e.g. `ExpiresAt`, `Description`):

```go
import "github.com/paygentic/sdk-go/optionalnullable"

optionalnullable.NewOptionalNullable(types.MustTimeFromString("2027-01-01T00:00:00Z"))
```
