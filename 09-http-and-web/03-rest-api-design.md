# REST API Design

## REST Principles

REST (Representational State Transfer) is an architectural style for HTTP APIs. It is not a protocol -- it is a set of **constraints**:

1. **Client-Server** -- separation of concerns
2. **Stateless** -- each request contains everything needed
3. **Cacheable** -- responses can be cached
4. **Uniform Interface** -- standardized interface (URLs, methods, status codes)
5. **Layered System** -- intermediaries (proxy, gateway) are transparent

## URLs (resources, not actions)

```
# GOOD — plural nouns
GET    /api/orders          # list
GET    /api/orders/42       # get by ID
POST   /api/orders          # create
PUT    /api/orders/42       # full update
PATCH  /api/orders/42       # partial update
DELETE /api/orders/42       # remove

# BAD — verbs in the URL
POST   /api/createOrder
GET    /api/getOrder/42
POST   /api/deleteOrder/42
```

### Nested resources

```
GET /api/customers/5/orders         # orders for customer 5
GET /api/customers/5/orders/42      # order 42 for customer 5
```

> Do not nest more than 2 levels -- it gets confusing. Prefer query parameters.

## Status Codes

### Success

| Code | Meaning | When to use |
|------|---------|-------------|
| **200** | OK | Successful GET, PUT, PATCH |
| **201** | Created | POST that created a resource. **Must** include a `Location` header with the new resource URI |
| **202** | Accepted | Request accepted for **asynchronous** processing (return a status URL the client can poll) |
| **204** | No Content | DELETE or PUT with no response body |

### Redirection

| Code | Meaning | When to use |
|------|---------|-------------|
| **304** | Not Modified | Conditional GET (`If-None-Match` / `If-Modified-Since`) — the client's cached copy is still valid |

### Client error

| Code | Meaning | When to use |
|------|---------|-------------|
| **400** | Bad Request | Validation failed, invalid body |
| **401** | Unauthorized | Not authenticated |
| **403** | Forbidden | Authenticated but without permission |
| **404** | Not Found | Resource does not exist |
| **409** | Conflict | Conflict (e.g., duplicate, concurrency) |
| **410** | Gone | Resource existed but was intentionally removed and will not come back |
| **415** | Unsupported Media Type | Server does not accept the request's `Content-Type` |
| **422** | Unprocessable Entity | Syntactically valid but semantically invalid |
| **429** | Too Many Requests | Rate limit hit. Include a `Retry-After` header |

### Server error

| Code | Meaning |
|------|---------|
| **500** | Internal Server Error |
| **502** | Bad Gateway |
| **503** | Service Unavailable |
| **504** | Gateway Timeout |

## Pagination

```
GET /api/orders?page=2&pageSize=20

Response:
{
  "data": [...],
  "page": 2,
  "pageSize": 20,
  "totalCount": 150,
  "totalPages": 8
}
```

### Keyset pagination (better performance)

```
GET /api/orders?after=order_xyz&limit=20
```

More performant than OFFSET for large datasets.

## Filters, sorting, and search

```
GET /api/orders?status=approved&customerId=5     # filter
GET /api/orders?sort=createdDate:desc            # sorting
GET /api/orders?search=notebook                  # text search
```

## Versioning

```
# Via URL (most common)
GET /api/v1/orders
GET /api/v2/orders

# Via header
GET /api/orders
Accept: application/vnd.myapi.v2+json

# Via query string
GET /api/orders?api-version=2.0
```

> URL prefix (`/v1/`) is the most pragmatic and easiest to route.

## Standardized error responses

Use the **RFC 7807 (Problem Details)** standard:

```json
{
  "type": "https://api.example.com/errors/validation",
  "title": "Validation error",
  "status": 400,
  "detail": "The 'name' field is required",
  "instance": "/api/orders",
  "errors": {
    "name": ["The 'name' field is required"],
    "email": ["Invalid email"]
  }
}
```

In ASP.NET Core:

```csharp
builder.Services.AddProblemDetails();

// Automatically returns Problem Details for errors
app.UseExceptionHandler();
app.UseStatusCodePages();
```

## Idempotency Keys

`POST` is not idempotent, but retries are unavoidable (network timeouts, mobile clients, queue consumers). **Idempotency keys** make retries safe without creating duplicate resources.

**How it works:**

1. The **client** generates a unique key (usually a UUID) per logical operation.
2. The client sends it in the `Idempotency-Key` HTTP header.
3. The **server** stores the response keyed by `(client, key)` for a TTL (e.g., 24h).
4. If the same key arrives again, the server returns the **cached response** instead of executing the operation again.

```http
POST /api/payments HTTP/1.1
Idempotency-Key: 8c1b0e2a-5f4d-4b3a-9d2e-1f0b3a4c5d6e
Content-Type: application/json

{ "orderId": 42, "amount": 150.00 }
```

```
First request  → 201 Created, charges the card, stores response under the key
Retry (same key) → 201 Created, returns the stored response (no new charge)
Different key  → treated as a new operation
```

> Popularized by Stripe. Standard pattern for payment, order, and money-movement APIs.

## HATEOAS (hyperlinks in the response)

```json
{
  "id": 42,
  "status": "pending",
  "total": 150.00,
  "_links": {
    "self": { "href": "/api/orders/42" },
    "approve": { "href": "/api/orders/42/approve", "method": "POST" },
    "cancel": { "href": "/api/orders/42/cancel", "method": "POST" },
    "customer": { "href": "/api/customers/5" }
  }
}
```

> In practice, few projects implement full HATEOAS. But it is good to know.

## Best practices

1. **Nouns** in the URL, **verbs** in HTTP methods
2. **Plural** for collections (`/orders`, not `/order`)
3. **Correct status codes** -- do not return 200 for everything
4. **Pagination** on listings -- never return all records
5. **Versioning** from the start
6. **Problem Details** for errors
7. **Idempotency** -- PUT and DELETE must be idempotent
8. **HTTPS** always
9. **Rate limiting** on public APIs
10. **Documentation** -- OpenAPI/Swagger

---

[← Previous: MIME Types](02-mime-types.md) | [Next: Web Security →](04-web-security.md) | [Back to index](README.md)
