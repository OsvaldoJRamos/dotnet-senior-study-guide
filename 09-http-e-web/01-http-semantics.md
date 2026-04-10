# HTTP Semantics

## HTTP Methods and Idempotency

**Idempotency**: applying the same operation multiple times produces the same result as the first application.

| Method | Safe | Idempotent | Description |
|--------|------|------------|-------------|
| `GET` | Yes | Yes | Fetch resource. Should **never** change server state |
| `POST` | No | No | Create resource or execute procedure |
| `PUT` | No | Yes | Replace entire resource. Client can generate the ID |
| `PATCH` | No | No* | Partially update a resource |
| `DELETE` | No | Yes | Remove resource. **Never return 404** when not found |

> *PATCH can be idempotent depending on the implementation

### DELETE and query parameters

Avoid query parameters in DELETE as much as possible. If necessary, **validate that all were provided**.

## URL

```
scheme://host:port/path?query#fragment
https://api.exemplo.com:443/v1/clientes?ativo=true#resultados
```

### Security tips

- **Avoid sensitive data in the URL** (email, SSN, data that identifies users)
- The URL is logged in many places (access logs, proxies, browser history)
- Query string is less frequently logged, but can still be captured

## TCP vs UDP

| Aspect | TCP | UDP |
|--------|-----|-----|
| Delivery guarantee | Yes | No |
| Packet ordering | Guaranteed | Not guaranteed |
| Speed | Slower | Faster |
| Usage | HTTP, APIs, email | Video, gaming, DNS |

## URL Encoding

Different parts of the URL have different types of encoding:

- `/` in the path is a separator, but in the query string it is normal
- `%` can be a problem in the query string, but not in the path
- Special characters must be encoded: `%20` (space), `%2F` (/), etc.

## Redirects

| Code | Type | Note |
|------|------|------|
| 301 | Permanent Redirect | Security issues in some scenarios |
| 302 | Found (temporary) | Historically inconsistent usage |
| 303 | See Other | Security issues |
| **307** | Temporary Redirect | Safe replacement for 302 |
| **308** | Permanent Redirect | Safe replacement for 301 |

> Prefer **307** and **308** as they are safer and more predictable.

## Content Negotiation

Client and server negotiate the content format:

```http
# Client says what it accepts
Accept: application/json

# Server says what it returns
Content-Type: application/json
```

---

[Next: MIME Types →](02-mime-types.md) | [Back to index](README.md)
