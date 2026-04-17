# Web Storage

The browser gives you four-and-a-half distinct places to persist data on the client: **cookies**, **localStorage**, **sessionStorage**, **IndexedDB**, and the **Cache Storage API**. Picking the wrong one is a security hole, a performance bug, or both. This page is about choosing correctly.

## Cookies

A cookie is a small key/value pair the browser automatically attaches to HTTP requests to the origin that set it. They predate every other storage API and are the only mechanism the browser sends to the server on its own.

### `Set-Cookie` attributes you must know

```http
Set-Cookie: sid=a3fWa; Max-Age=2592000; Path=/; Secure; HttpOnly; SameSite=Lax
```

| Attribute | What it does |
|-----------|--------------|
| `Secure` | Cookie is sent only over HTTPS (`except on localhost`). |
| `HttpOnly` | JavaScript cannot read it via `document.cookie`. Mitigates XSS token theft. |
| `SameSite=Strict` | Sent only on same-site requests — use for auth / cart cookies. |
| `SameSite=Lax` | Sent on same-site and on top-level cross-site navigations using a safe method (`GET`/`HEAD`). Used as the default by some browsers when the attribute is omitted — don't rely on it, set it explicitly. |
| `SameSite=None` | Sent on all cross-site requests. **Requires `Secure`.** Use for third-party embeds. |
| `Path` | Cookie is sent only if the request URL path matches this prefix. |
| `Domain` | Makes the cookie available to the given host and its subdomains. Omitting it is stricter (host-only cookie). |
| `Max-Age` | Lifetime in seconds. Takes precedence over `Expires` when both are set. |
| `Expires` | Absolute HTTP-date. Without `Max-Age` or `Expires`, the cookie is a session cookie. |
| `Partitioned` | Opts the cookie into CHIPS partitioned storage. Requires `Secure`. |

> Per MDN, SameSite defaults: "Some browsers use `Lax` as the default value if `SameSite` is not specified." Don't rely on it — set it explicitly.

### SameSite in one sentence each

- **`Strict`** — never sent on cross-site requests, not even top-level navigation from another site. Safest, but can break "click this email link and land logged in".
- **`Lax`** — sent on top-level navigation with safe methods (GET). Not sent on cross-site `POST`/`PUT`/`DELETE` or on subresource requests (iframes, images, fetch).
- **`None; Secure`** — unrestricted, cross-site sends allowed. Required for third-party iframes that need their own cookies (embedded chat, payments, analytics).

### CSRF implications

CSRF exists because the browser automatically attaches cookies to cross-site requests. `SameSite=Lax` (used as the default by some browsers — set it explicitly anyway) blocks the dangerous vector — cross-site `POST` from `evil.com` to `bank.com` won't include the session cookie. `SameSite=Strict` closes it fully. For APIs used by a SPA on the same site, `Lax` + CSRF token for state-changing requests is the conservative combo.

### Cookie name prefixes: `__Host-` and `__Secure-`

These aren't just cosmetic — browsers enforce constraints when the name starts with them:

- **`__Secure-`** — "must be set with the `Secure` attribute by a secure page (HTTPS)."
- **`__Host-`** — same as `__Secure-`, **plus** "they must not have a `Domain` attribute specified, and the `Path` attribute must be set to `/`." This guarantees the cookie is bound to the exact host that set it and cannot be overwritten by a sibling subdomain.

```http
Set-Cookie: __Host-sid=a3fWa; Path=/; Secure; HttpOnly; SameSite=Lax
```

> Use `__Host-` for session cookies whenever you can. A compromised subdomain (e.g., `uploads.example.com`) cannot clobber a `__Host-sid` cookie set by `example.com`.

### Size

Per MDN: "Browsers are generally limited to a maximum number of cookies per domain (varies by browser, generally in the hundreds), and a maximum size per cookie (usually 4KB)." Treat ~4 KB per cookie as the budget and don't jam blobs or JWTs with 20 claims into them.

### CHIPS (Partitioned cookies)

With third-party cookies being phased out, `Partitioned` (CHIPS — "Cookies Having Independent Partitioned State") lets a third-party cookie still work in an embed, but keyed by the top-level site so it can't be used for cross-site tracking.

```http
Set-Cookie: __Host-embed=34d8g; SameSite=None; Secure; Path=/; Partitioned
```

MDN: "Cookies marked `Partitioned` are double-keyed: by the origin that sets them _and_ the origin of the top-level page." Each top-level site gets its own isolated cookie jar for that embed. Must be set with `Secure`.

### Consent

Anything beyond strictly-necessary session cookies typically needs consent under GDPR / ePrivacy. The engineering consequence: gate non-essential `Set-Cookie` headers and analytics SDKs behind a consent signal — don't set them at page load by default.

## `localStorage` and `sessionStorage`

Synchronous string-only key/value stores, same-origin, accessed via JavaScript only (the server never sees them).

```js
localStorage.setItem('theme', 'dark');
const theme = localStorage.getItem('theme');
localStorage.removeItem('theme');

sessionStorage.setItem('wizard-step', '3');
```

### Differences

| | `localStorage` | `sessionStorage` |
|---|----------------|------------------|
| Lifetime | Persists across tabs and restarts | "Closing the browser tab destroys all `sessionStorage` data associated with that tab." |
| Partitioning | "partitioned by origin only" | "partitioned by browser tabs and by origin" |
| Cross-tab visibility | Shared across tabs of the same origin | Isolated per tab |
| `storage` event | Fires on OTHER tabs/windows of the same origin when a key changes | Does not cross tabs |

### Synchronous API is a footgun

MDN: "Both `sessionStorage` and `localStorage` in Web Storage are synchronous in nature... blocking the execution of other JavaScript code until the operation is completed." Every read/write is main-thread blocking. Writing a large JSON blob on every keystroke will jank your UI.

### Gotchas

- **Strings only.** You end up calling `JSON.stringify` / `JSON.parse` everywhere, paying the serialization cost on each access.
- **No expiry.** Values live forever unless you delete them or encode your own TTL alongside the value.
- **Quota.** MDN doesn't pin a specific number for Web Storage on the main API page and points to "Storage quotas and eviction criteria" for details. Browser implementations typically allow around 5–10 MB per origin, but don't hard-code that — wrap writes in `try/catch` for `QuotaExceededError`.
- **Not for secrets.** Accessible to any script running on the page, including anything XSS injects.

### Cross-tab sync with `storage` event

```js
window.addEventListener('storage', (e) => {
  if (e.key === 'auth') {
    // Another tab logged out — react here.
  }
});
```

Fires only in *other* tabs of the same origin, not the one that made the change.

## IndexedDB

> "IndexedDB is a low-level API for client-side storage of significant amounts of structured data, including files/blobs. This API uses indexes to enable high-performance searches of this data."

Asynchronous, transactional, keyed object stores with secondary indexes. This is the right choice when you hit the limits of `localStorage` (size, sync API, no indexes).

### Minimal example

```js
const req = indexedDB.open('app', 2);

req.onupgradeneeded = (event) => {
  const db = event.target.result;
  if (!db.objectStoreNames.contains('orders')) {
    const store = db.createObjectStore('orders', { keyPath: 'id' });
    store.createIndex('by_customer', 'customerId', { unique: false });
  }
};

req.onsuccess = (event) => {
  const db = event.target.result;
  const tx = db.transaction('orders', 'readwrite');
  tx.objectStore('orders').put({ id: 'o-1', customerId: 'c-42', total: 99 });
  tx.oncomplete = () => console.log('saved');
};
```

- **Transactions** — "You create a transaction on a database, specify the scope... and determine the kind of access (read only or readwrite) that you want." Reads/writes happen inside transactions; the transaction auto-commits when control returns to the event loop.
- **Object stores** hold records keyed by a primary key.
- **Indexes** allow fast lookup by non-primary fields without scanning every record.
- **Versioning** — you bump the integer version when the schema changes and migrate inside `onupgradeneeded`. There is no ALTER; you add/remove stores and indexes there.

### When to use

- Offline-first apps (PWAs) that need durable local state.
- Large structured data (message history, cached API data beyond `localStorage` limits, binary blobs).
- Anything where you'd reach for a client-side database.

### Libraries

The raw API is event-based and verbose. In practice, reach for a wrapper:

- **idb** — a tiny Promise wrapper over the native API, keeps you close to the metal.
- **Dexie.js** — higher-level with a query builder and reactivity.

> Rule of thumb: if you're writing more than a few hundred lines against raw IndexedDB, switch to `idb` or Dexie.

## Cache Storage API

Not to be confused with the HTTP cache. The `Cache` interface is a programmable key/value store of `Request` → `Response` pairs, defined alongside Service Workers but usable from window scope too.

MDN: "The `Cache` interface provides a persistent storage mechanism for `Request` / `Response` object pairs that are cached in long lived memory."

Critical distinction — it is independent from the browser's HTTP cache:

> "The caching API doesn't honor HTTP caching headers."

That means `Cache-Control: max-age=3600` is ignored. Entries live until you delete them: "Items in a `Cache` do not get updated unless explicitly requested; they don't expire unless deleted."

### Typical use from a Service Worker

```js
// In service-worker.js
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open('v1').then(async (cache) => {
      const hit = await cache.match(event.request);
      if (hit) return hit;
      const resp = await fetch(event.request);
      cache.put(event.request, resp.clone());
      return resp;
    })
  );
});
```

Use it for offline app shells, precached static assets, and custom caching strategies (stale-while-revalidate, network-first) that the HTTP cache can't express.

## Storage quota and eviction

All of the above (except cookies) share a per-origin quota governed by the Storage Standard. You don't control the exact number — you ask the browser.

```js
const { usage, quota } = await navigator.storage.estimate();
console.log(`Using ${usage} of ${quota} bytes`);
```

Per MDN: `navigator.storage.estimate()` "returns a promise that, when resolved, receives an object that contains these figures." Numbers are estimates, not exact.

### Best-effort vs persistent

By default, storage is **best-effort** — the browser may evict it under pressure without warning. You can request **persistent** storage:

```js
if (await navigator.storage.persist()) {
  // Won't be evicted automatically — user must clear it explicitly.
}
```

MDN on best-effort: "The user agent will try to retain the data contained in the bucket for as long as it can, _but will not warn users_ if storage space runs low and it becomes necessary to clear the bucket."

MDN on persistent: "The user agent will retain the data as long as possible, clearing all `"best-effort"` buckets before considering clearing a bucket marked `"persistent"`."

> Request persistence only for data whose loss would visibly break the app (offline docs, unsynced drafts). Persistent storage typically requires user permission.

## Security: where tokens live

Every time you put an auth token in the browser, pick between two realistic options:

| Location | XSS steals it? | Automatic on requests? | Verdict |
|----------|----------------|------------------------|---------|
| `localStorage` / `sessionStorage` | **Yes** — any injected script reads it | No (you attach it manually) | Avoid for session tokens. |
| Cookie with `HttpOnly; Secure; SameSite=Lax` (or `Strict`) | No — JS cannot read it | Yes (browser attaches it) | Recommended for session identifiers. |

OWASP HTML5 Security Cheat Sheet: "Do not store session identifiers in local storage as the data is always accessible by JavaScript." And: "Cookies can mitigate this risk using the `httpOnly` flag."

MDN on `HttpOnly`: "A cookie with the `HttpOnly` attribute can't be accessed by JavaScript... This precaution helps mitigate cross-site scripting (XSS) attacks."

> A JWT in `localStorage` is one XSS away from account takeover. A JWT in an `HttpOnly; Secure; __Host-` cookie with `SameSite=Lax` + CSRF protection survives client-side XSS of the token itself. Trade-off: the server manages the cookie, and cross-site SPA scenarios need CORS + `SameSite=None; Secure`.

### Third-party cookies and the Privacy Sandbox

Chromium and other browsers have been phasing out unrestricted third-party cookies. If you own an embed that needs its own cookie, set it as `SameSite=None; Secure; Partitioned` (CHIPS) so it keeps working in a partitioned cookie jar without enabling cross-site tracking.

## Comparison table

| | Cookies | `localStorage` | `sessionStorage` | IndexedDB | Cache Storage |
|---|---------|----------------|------------------|-----------|---------------|
| Size budget | ~4 KB per cookie | Typically ~5–10 MB per origin (browser-dependent) | Typically ~5–10 MB per origin (browser-dependent) | Large — governed by Storage Standard quota | Large — governed by Storage Standard quota |
| Scope | Origin (with `Path`/`Domain` rules) | Origin | Origin + tab | Origin | Origin |
| Lifetime | Until `Expires`/`Max-Age` or session ends | Until explicitly cleared | Until tab closes | Until explicitly cleared or evicted | Until explicitly deleted |
| API | HTTP headers + `document.cookie` (unless `HttpOnly`) | Synchronous JS | Synchronous JS | Asynchronous, transactional JS | Asynchronous (Promises) |
| Sent with HTTP requests? | Yes, automatically | No | No | No | No |
| Readable by JS? | Only if no `HttpOnly` | Yes | Yes | Yes | Yes |
| Good for | Session identifiers, CSRF tokens, short server-side flags | Non-sensitive UI prefs (theme, language) | Per-tab ephemeral state (wizards) | Offline-first apps, large structured data, blobs | Programmable HTTP response cache, offline app shell |
| Avoid for | Large payloads, client-only data | Auth tokens, large data, anything sync-write-heavy | Auth tokens | Tiny simple key/value (overkill) | Regular data storage |

---

[← Previous: iframes and Embedding](04-iframes-and-embedding.md) | [Next: Web Performance →](06-web-performance.md) | [Back to index](README.md)
