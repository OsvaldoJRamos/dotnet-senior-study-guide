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
GET    /api/pedidos          # list
GET    /api/pedidos/42       # get by ID
POST   /api/pedidos          # create
PUT    /api/pedidos/42       # full update
PATCH  /api/pedidos/42       # partial update
DELETE /api/pedidos/42       # remove

# BAD — verbs in the URL
POST   /api/criarPedido
GET    /api/obterPedido/42
POST   /api/deletarPedido/42
```

### Nested resources

```
GET /api/clientes/5/pedidos         # orders for customer 5
GET /api/clientes/5/pedidos/42      # order 42 for customer 5
```

> Do not nest more than 2 levels -- it gets confusing. Prefer query parameters.

## Status Codes

### Success

| Code | Meaning | When to use |
|------|---------|-------------|
| **200** | OK | Successful GET, PUT, PATCH |
| **201** | Created | POST that created a resource |
| **204** | No Content | DELETE or PUT with no response body |

### Client error

| Code | Meaning | When to use |
|------|---------|-------------|
| **400** | Bad Request | Validation failed, invalid body |
| **401** | Unauthorized | Not authenticated |
| **403** | Forbidden | Authenticated but without permission |
| **404** | Not Found | Resource does not exist |
| **409** | Conflict | Conflict (e.g., duplicate, concurrency) |
| **422** | Unprocessable Entity | Semantically invalid |

### Server error

| Code | Meaning |
|------|---------|
| **500** | Internal Server Error |
| **502** | Bad Gateway |
| **503** | Service Unavailable |
| **504** | Gateway Timeout |

## Pagination

```
GET /api/pedidos?page=2&pageSize=20

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
GET /api/pedidos?after=pedido_xyz&limit=20
```

More performant than OFFSET for large datasets.

## Filters, sorting, and search

```
GET /api/pedidos?status=aprovado&clienteId=5     # filter
GET /api/pedidos?sort=dataCriacao:desc            # sorting
GET /api/pedidos?search=notebook                  # text search
```

## Versioning

```
# Via URL (most common)
GET /api/v1/pedidos
GET /api/v2/pedidos

# Via header
GET /api/pedidos
Accept: application/vnd.minhaapi.v2+json

# Via query string
GET /api/pedidos?api-version=2.0
```

> URL prefix (`/v1/`) is the most pragmatic and easiest to route.

## Standardized error responses

Use the **RFC 7807 (Problem Details)** standard:

```json
{
  "type": "https://api.exemplo.com/errors/validation",
  "title": "Erro de validação",
  "status": 400,
  "detail": "O campo 'nome' é obrigatório",
  "instance": "/api/pedidos",
  "errors": {
    "nome": ["O campo 'nome' é obrigatório"],
    "email": ["Email inválido"]
  }
}
```

In ASP.NET Core:

```csharp
builder.Services.AddProblemDetails();

// Retorna automaticamente Problem Details para erros
app.UseExceptionHandler();
app.UseStatusCodePages();
```

## HATEOAS (hyperlinks in the response)

```json
{
  "id": 42,
  "status": "pendente",
  "total": 150.00,
  "_links": {
    "self": { "href": "/api/pedidos/42" },
    "aprovar": { "href": "/api/pedidos/42/aprovar", "method": "POST" },
    "cancelar": { "href": "/api/pedidos/42/cancelar", "method": "POST" },
    "cliente": { "href": "/api/clientes/5" }
  }
}
```

> In practice, few projects implement full HATEOAS. But it is good to know for interviews.

## Best practices

1. **Nouns** in the URL, **verbs** in HTTP methods
2. **Plural** for collections (`/pedidos`, not `/pedido`)
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
