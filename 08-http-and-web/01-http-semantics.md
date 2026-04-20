# HTTP Semantics

## HTTP Methods and Idempotency

**Idempotency**: applying the same operation multiple times produces the same result as the first application.

| Method | Safe | Idempotent | Description |
|--------|------|------------|-------------|
| `GET` | Yes | Yes | Fetch resource. Should **never** change server state |
| `POST` | No | No | Create resource or execute procedure |
| `PUT` | No | Yes | Replace entire resource. Client can generate the ID |
| `PATCH` | No | No* | Partially update a resource |
| `DELETE` | No | Yes | Remove resource (see note on 404 vs 204 below) |

> *PATCH can be idempotent depending on the implementation

### DELETE when the resource is missing

Both responses are valid per RFC 9110:

- **404 Not Found** — correct: the resource does not exist.
- **204 No Content** — also correct: some APIs always return 204 on DELETE, treating the operation as idempotent from the client's point of view (repeated deletes look identical).

> Pick one convention and **document it**. What matters is consistency across the API, not which code you pick.

### DELETE and query parameters

Avoid query parameters in DELETE as much as possible. If necessary, **validate that all were provided**.

## URL

```
scheme://host:port/path?query#fragment
https://api.example.com:443/v1/customers?active=true#results
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

The real distinction between the 3xx codes is **method preservation** — what the client does with the method and body on the redirected request.

| Code | Type | Method/body on redirect |
|------|------|-------------------------|
| 301 | Moved Permanently | Legacy: browsers may rewrite POST → GET |
| 302 | Found (temporary) | Legacy: browsers may rewrite POST → GET |
| 303 | See Other | **Must switch to GET** — by design |
| **307** | Temporary Redirect | **Method and body preserved** |
| **308** | Permanent Redirect | **Method and body preserved** |

### When to use which

- **303 See Other** — the canonical answer to **POST-Redirect-GET**. After a successful POST, redirect to a GET URL so the browser loads a resource page (no resubmission on refresh).
- **307 / 308** — when you need the same method/body replayed against a new URL (e.g., moving an API endpoint without forcing clients to rebuild POST bodies).
- **301 / 302** — historically ambiguous because of the legacy POST→GET behavior. Prefer **308** over 301 and **307** over 302 when method preservation matters.

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
