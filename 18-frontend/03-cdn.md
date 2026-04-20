# Content Delivery Networks (CDN)

A **CDN** is a geographically distributed network of caching servers (edge locations, POPs) that sit between your users and your **origin** server. The goal is simple: serve bytes from a location close to the user, with a warm cache, so the origin sees less traffic and the user sees lower latency.

In interviews, if you say "we put it behind a CDN", expect follow-ups on `Cache-Control` directives, validators, cache keys, and invalidation. Those are the real topics.

## Edge and origin

| Term | Meaning |
|------|---------|
| **Origin** | Your actual server (app, S3 bucket, object store). Source of truth. |
| **Edge / POP** | A caching server close to users. Hundreds globally on large CDNs. |
| **Cache hit** | Edge serves the response directly. Origin is not contacted. |
| **Cache miss** | Edge has no valid copy. It fetches from origin (or a mid-tier / shield). |
| **Origin Shield / tiered cache** | An extra cache tier between edges and origin to consolidate requests. |

### Pull vs push

| Model | How it works | When to use |
|-------|--------------|-------------|
| **Pull-through** (default) | CDN fetches from origin on first miss and caches the response. | Almost always. Any public HTTP asset. |
| **Push** | You upload assets directly to the CDN's storage; no origin pull. | Large pre-rendered video catalogs, one-off campaigns, origin you don't want exposed. |

> Default to pull. It's simpler, cache-warming happens naturally with real traffic.

## Cache-Control directives that actually matter

These are defined by [RFC 9111](https://datatracker.ietf.org/doc/html/rfc9111) and the `stale-*` extensions by [RFC 5861](https://datatracker.ietf.org/doc/html/rfc5861).

| Directive | Applies to | What it does |
|-----------|------------|--------------|
| `max-age=N` | browser + CDN | Response is fresh for N seconds (MDN: "remains fresh until N seconds after the response is generated"). |
| `s-maxage=N` | shared caches only (CDN) | Overrides `max-age` for shared caches. Ignored by browsers. |
| `public` | shared caches | Response may be stored in a shared cache. Needed to cache responses with `Authorization`. |
| `private` | browser only | Shared caches **MUST NOT** store. Use for user-personalized responses. |
| `no-cache` | both | May be stored, but **must revalidate** with origin before reuse. |
| `no-store` | both | Do not store, anywhere, ever. |
| `must-revalidate` | both | Once stale, MUST NOT serve without successful validation. |
| `immutable` | both | Response will not change while fresh. Browser will skip revalidation on refresh. |
| `stale-while-revalidate=N` | both | Serve stale up to N seconds while revalidating in background. |
| `stale-if-error=N` | both | Serve stale up to N seconds if origin returns 5xx (500, 502, 503, 504). |

### Hashed static assets (JS/CSS bundles)

```http
Cache-Control: public, max-age=31536000, immutable
```

Filename contains the hash (`app.a1b2c3.js`), so "update" = new filename. `immutable` tells the browser "don't bother revalidating on reload" (MDN: "indicates that the response will not be updated while it's fresh").

### HTML pages

```http
Cache-Control: public, max-age=0, s-maxage=60, stale-while-revalidate=300
```

Browser always revalidates, CDN serves fresh for 60s and stale-while-revalidating for 5 more minutes. Users see fast responses, origin sees one request per minute per POP.

### Private user content

```http
Cache-Control: private, no-store
```

Anything tied to `Authorization` or Set-Cookie that must not leak across users. `no-store` is the safe default when in doubt.

> Shared caches MUST NOT reuse a cached response for a request with `Authorization` unless the response directives explicitly allow it (RFC 9111). `public` opts in.

## Validators and conditional requests

Freshness expires. Validators let you revalidate cheaply.

### `ETag` / `If-None-Match`

```http
# First response
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"

# Subsequent request
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"

# If unchanged
HTTP/1.1 304 Not Modified
```

A 304 has no body — just headers. You save bandwidth, not round-trips.

**Strong vs weak ETags:** `W/"abc"` is weak (semantically equivalent, not byte-identical). Weak ETags disable byte-range caching (MDN: "weak ETags prevent caching when byte range requests are used").

### `Last-Modified` / `If-Modified-Since`

Fallback validator for when you don't have an ETag. Coarser (1-second resolution), but better than nothing.

> Prefer `ETag`. Use `Last-Modified` only when content has a real timestamp you trust.

## Cache key and `Vary`

The **cache key** is what the CDN uses to look up a cached response. By default it's `method + URL`. You can extend it with `Vary`.

```http
Vary: Accept-Encoding
```

Means "cache a separate entry per `Accept-Encoding` value". One entry for `gzip`, another for `br`.

> MDN: "Including a `Vary` header ensures that responses are separately cached based on the headers listed."

### The Vary explosion

Every value you list multiplies the number of cache entries:

- `Vary: Accept-Encoding` → ~3 variants (gzip, br, identity). Fine.
- `Vary: User-Agent` → thousands of variants. **Cache hit rate collapses.** Never do this.
- `Vary: Cookie` without normalization → one entry per user. Effectively no caching.

`Vary: *` tells caches the response is effectively uncacheable (MDN: "Implies that the response is uncacheable.").

**Fix:** if personalization is required, either move the personalized bits to a separate request, or normalize the varying header at the edge (e.g., bucket User-Agent into `mobile` / `desktop`).

## Invalidation and purge

Cache invalidation is a hard problem. Options:

| Strategy | How it works | Trade-off |
|----------|--------------|-----------|
| **URL-based purge** | Purge `https://cdn.example.com/foo.json` | Precise, but doesn't scale across many keys. |
| **Tag-based purge** (surrogate keys) | Tag responses (`Surrogate-Key: product-123`), purge by tag | Best for related content. Vendor-specific header names. |
| **Soft purge** | Mark stale instead of deleting; `stale-while-revalidate` covers the gap | Origin is never hit by a thundering herd. |
| **Cache-busting via URL hash** | `app.a1b2c3.js` becomes `app.d4e5f6.js` | No purge needed. Preferred for static assets. |

> For frequently changing content, prefer **short TTL + stale-while-revalidate** over aggressive purging. Purge APIs have rate limits and propagation delay.

## SSL/TLS termination at the edge

CDNs typically terminate TLS at the edge, then re-encrypt to the origin over a separate connection. Benefits:

- Lower handshake latency (TLS completes closer to the user).
- Centralized certificate management (ACM, Cloudflare SSL).
- DDoS mitigation at the edge before traffic hits origin.

Trade-off: the CDN sees plaintext. Most enterprises are OK with this; regulated industries may not be.

## Geographic routing

Users hit the nearest POP via **anycast IP** — the same IP is advertised from many locations, BGP routes each request to the closest one. This is built into every major CDN.

Some products add **geo steering at the app layer** (route EU users to an EU origin) via `CF-IPCountry` / `CloudFront-Viewer-Country` headers.

## Static assets vs API responses

| Aspect | Static assets | API responses |
|--------|---------------|---------------|
| Hit rate | Very high | Variable; often low |
| TTL | Long (year) with `immutable` | Short (seconds to minutes) |
| Key strategy | URL + hash in filename | URL + query + a few `Vary` headers |
| Purge need | Rare (use hashed names) | Frequent (tag-based ideal) |
| Personalization | None | Often — use `private` or vary carefully |

> Caching `GET` JSON APIs at the CDN is high-leverage. Caching anything with `Authorization` requires `s-maxage` + `public` and a very clear mental model, or a normalized cache key. When unsure, don't cache.

## Common gotchas

- **Double caching.** Origin sets `Cache-Control: max-age=60` but your CDN has a 1-hour rule. Users see stale content for an hour. Decide who owns TTLs.
- **Vary explosion.** See above. Audit every `Vary` header.
- **CORS + cache.** If the response varies by `Origin` for CORS, you need `Vary: Origin`. Forgetting this means one caller's CORS headers leak to another.
- **Cookies in the cache key.** Default-stripping cookies is the right move for public assets. Keep them only where they affect the response.
- **Auth headers and shared caches.** RFC 9111: shared caches MUST NOT reuse responses to `Authorization` requests unless explicitly allowed. Adding `public` opts in — make sure that's intentional.
- **`no-cache` is not "do not cache".** MDN: "`no-cache` allows caches to store a response but requires them to revalidate it before reuse." For "do not cache", use `no-store`.

## Major CDNs at a glance

| CDN | Notable strength | Notes |
|-----|------------------|-------|
| **Cloudflare** | Global network, integrated security (WAF, Bot Management), Workers at edge. Has **Tiered Cache** for mid-tier caching. | Flat pricing tiers; generous free tier. |
| **Fastly** | Programmable VCL, fast purge (sub-second), strong for news/media. | Heavier learning curve; pay-as-you-go. |
| **Akamai** | Largest footprint, enterprise-grade features, heavy in media/enterprise. | Expensive; complex config. |
| **AWS CloudFront** | Deep AWS integration (S3, Lambda@Edge, WAF), **Origin Shield** feature. | Good default when already on AWS. |
| **Azure Front Door** | Azure-native, integrated WAF and global load balancing. | Good default when on Azure. |

> Don't pick a CDN on features alone. Pick on: (1) where your users are, (2) where your origin is, (3) what ecosystem (AWS/Azure/GCP) you're in, (4) your purge and log requirements.

---

[← Previous: DOM and Events](02-dom-and-events.md) | [Next: iframes and Embedding →](04-iframes-and-embedding.md) | [Back to index](README.md)
