# ASP.NET Core Best practices

> A collection of best practices for ASP.NET Core Web API/App design. Short, brief and effective.

## API design

* Design API from consumer perspective, not DB perspective. If you are selling books, you should have Get Books, not Get Products.
* Avoid Minimal Value Product & versioning thinking.

### Avoid `bool`s

* Locking into two states only.

```json
{
    "isVisible": true
}
```

More flexible design

```json
{
    "visibility": "visible"
}
```

### Primary key in DTO

* Id in DTO should not be a primary key from DB, but some unique, but randomly generated string, possible human readable (slug).

```json
{
    "id": 567
}
```

More secure design

```json
{
    "id": "the-matrix-1999"
}
```

### Fundamental data types obsession

* Compound objects are OK for DTOs.

```json
{
    "street": "Highway street",
    "postalCode": 50002,
    "city": "Hradec Králové",
    "country": "CZE"
}
```

Improved design

```json
{
    "id": "asdfasdfasdf",
    "name": "kutlime",
    "address": {
        "street": "Highway street",
        "postalCode": "50002",
        "city": "Hradec Králové",
        "country": "CZE"
    }
}
```

### Prevent versioning any case

* Always thinks ahead.

```csharp
id      int      167  -> After that, we switched to Guid. 🤷🏿‍♀️
price   decimal  0,9  -> After that, we started to support different currencies.
price   bool     true -> After that, we started to support partial payments.
color   string   red  -> After that, we needed hex code. 🤷🏿‍♀️
```

Improved design

```json
{
    "id": "the-matrix-1999",
    "title": "The Matrix",
    "price": {
        "base": "19",
        "decimals": "95",
        "currency": "USD"
    },
    "paymentState": "Paid",
    "Color": "ff00ff"
}
```

### Avoid stateful APIs

* ❌ Session cookie authentication.
* ❌ Agenda selection in first request, second request for data.
* ✅ Use custom headers for metadata.

### Use webhooks for client notifications

* ❌ Use pooling to deliver information to clients.
* ✅ Use webhooks.

### Wrong URL design and IDs

* ❌ Post `/invoices/set-paid/` (no resource identifier here).
* ❌ Put `/invoices/ {... "status": "paid"}` (no available resource for paid operation, need to take whole invoice).
* ✅ Post `/payments` with `{"invoiceId": 1, "amount": 123, "currency": "CZK, "transactionId": "bkykjsoanhkunapo"}`.
* ❌ Get `/invoices/1/header` (sequential ID and returning an instance from different entity, e.g., a header for an invoice).
* ✅ Get `/invoice-header/1`.
* ❌ Get `/customers/9/invoices` (filtering, sooner or later it will be updated with some filtering, paging, etc.).
* ✅ Get `/invoices/?customerId=1`.
* ❌ Get `/customers/+15551234567/` (sensitive information in URL, card numbers, etc.).
* ❌ Get `/campaigns/100/send` (execution of operation via Get request).
* ✅ Post `/distributions` with `{"campaignId": 100, "date": "2022-09-12"}`.

### Inconsistent naming

* ❌ `/users?state=active&location=praha` (inconsistent use of `=`, first works as equals, second as contains).
* ❌ `/users?registered=2023-01-01&approvedAt=2023-02-02` (inconsistent use suffix `at`).
* ✅ `/users?registeredAt=2023-01-01&approvedAt=2023-02-02`.
* ✅ `/users?registeredFrom=2023-01-01&approvedAt=2023-02-02`.
* ❌ `/posts?approved=true&unlisted=true` (combination of positive & negative conditions).
* ✅ `/posts?approved=true&listed=false`.
* ❌ `/webinar?duration=1` (ignoring standards for representation of various types of resources).
* ✅ `/webinar?duration=P1DT30H4S` (ISO 8601, RFC 3339).
* ❌ `/invoices?currency=1` (ignoring the standard).
* ✅ `/invoices?currency=CZK` (ISO 4217).

### 200 for everything

* ❌ HTTP/1.1 200 OK `{"status": 1, "warnings": [], "errors": [], "invoices": [{}], "totalItems": 123}` (inconsistent body, conditional parameters, returning null).
* ❌ HTTP/1.1 200 OK `{"status": 2, "warnings": [], "errors": [{"message": "bad filter type"}], "invoices": null, "trackingId": "kaezoynvblap"}`.
* ✅ Follow standards (RFC 7231).

### State code dependent error structures

* ❌ Different error structures based on the error code.
* ✅ Follow standards (RFC 7807) and use stable error structures with mapping appropriate properties to the error structure based on the error code.
* ❌ Unappropriated response type based on client request (a client requests XML, we return JSON)
* ✅ Use standardized response following RFC 7807 with static titles (no IDs or dynamic strings), type mapping

```http
HTTP/1:1 403 Forbidden
Content-Type: application/problem+json
Content-Language: en

{
    "type": "https://github.com/api/errors/insufficient-permissions",
    "title": "Insufficient permissions",
    "status": 403,
    "detail": "Insufficient permissions for changing resource.",
    "instance": "https://github.com/api/users/kutlime",
}
```

## Preferring documentation over specification

* ❌ The generated documentation for API has poor quality (tooling for generating documentation doesn't cover all OpenAPI features and isn't really sophisticated enough).
* ❌ Application must run somewhere to provide documentation.
* ❌ If you split the project, you need to merge it together.
* ❌ You **can't** develop FE & BE in parallel, because you have to program something to generate documentation.
* ✅ You **can** develop FE & BE in parallel with the specification first approach.
* ❌ Versions are part of the code, so a new version of code means deploy a new version of code.
* ✅ You can have as many specification as you want.
* ✅ Always design API with OpenAPI specification first, program later

## Validation fragmentation

The default behavior of .NET is automatic 1st level of validation via `ApiBehaviorOptions`, 2nd level of validation with `IValidatableObject`, 3rd level of validation as custom.

✅ All of this can be covered by one tool **FluentValidation**.

## Missing HTTP cache

* ❌ No resource caching on API.
* ❌ Only server-side cache validation.
* ❌ Static cache thanks to `max-age` with invalid data.
* ❌ Resource structure that cannot be cached.

The object can be cached due to property `totalBooks` that can change.

```json
{
    "title": "ASP.NET Core Best practice",
    "author": {
        "name": "Microsoft Team",
        "totalBooks": 42},
    "categories": [{"name": "IT"}]
}
```

## Missing hot path optimization

* ❌ No CQRS
* ❌ Ineffective validation first (calls to repository)
* ✅ Go from the most cheapest validation to the most expensive ones (typically DB calls).
* ❌ Premature optimization for edge cases.
* ✅ Mature optimization for hot paths, byznys as usual, cases.

## Missing support for Patch operation(s)

* ❌ Entity update via Put (whole entity must be provided).
* ❌ Large number of endpoint for every operation.
* ❌ Patch exists, but not following RFC 6902.
* ❌ Patch exists, uses RFC 6902, but defines custom operation (not allowed in RFC 6902).

## Not using cancellation token in async operation

* ❌ Well... Do I have to describe this somehow? 🤷🏿‍♀️

## Links

* [CZE] [Nejčastější nedostatky API](https://www.youtube.com/watch?v=gGeM1MX6Co8)
* [CZE] [Přerušení HTTP request a cancellation tokens](https://www.youtube.com/watch?v=tXgNHmp64e8)
