# PWA and Service Workers

A **Progressive Web App (PWA)** is a web app that can be installed to the device home screen and keep working offline. The two pieces that make it possible are the **Web App Manifest** (describes how it installs) and the **Service Worker** (makes it work offline and in the background).

## What makes a PWA

Per web.dev's [install criteria](https://web.dev/articles/install-criteria), Chrome considers a page installable when it meets all of:

- Served over **HTTPS** (or `localhost` for development — MDN calls this out as the explicit exception).
- Has a valid **Web App Manifest** that includes: `short_name` or `name`, `icons` (at minimum 192×192 and 512×512), `start_url`, a `display` value of `fullscreen`, `standalone`, `minimal-ui`, or `window-controls-overlay`, and no `prefer_related_applications: true`.
- Has a registered **Service Worker** (or, in some recent Chrome versions, this has been relaxed — verify against your target Chrome version; Firefox and Edge have their own variations).
- User engagement heuristics: at least one click/tap on the page and ~30 seconds of viewing time before the install prompt is offered.

> A service worker is *fundamental to offline PWA behavior* even where installability alone may no longer strictly require it. Always ship one.

## Web App Manifest

A JSON file linked from the `<head>`:

```html
<link rel="manifest" href="/manifest.webmanifest">
```

```json
{
  "name": "Order Tracker",
  "short_name": "Orders",
  "start_url": "/?source=pwa",
  "display": "standalone",
  "theme_color": "#0b5fff",
  "background_color": "#ffffff",
  "icons": [
    { "src": "/icons/192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/512-maskable.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

Served with MIME type `application/manifest+json`.

### Key members

| Member | Purpose |
|--------|---------|
| `name` | Full application name, shown during install and on the splash screen |
| `short_name` | Used under the home-screen icon where space is tight |
| `start_url` | URL loaded when launched from the installed icon — tagging it (`?source=pwa`) is useful for analytics |
| `display` | How the app is chromed when launched (see below) |
| `icons` | Array of icons — include 192 and 512, plus a `maskable` variant for Android |
| `theme_color` | Colors the OS UI (status bar, task switcher) |
| `background_color` | Splash-screen background while the app boots |

### `display` values

From MDN's display reference:

| Value | UI |
|-------|----|
| **`fullscreen`** | No browser UI, full screen. Good for games. |
| **`standalone`** | Looks like a native app window. No URL bar; status bar may remain. Most common PWA choice. |
| **`minimal-ui`** | App window with a minimal set of browser controls (back, reload). Good for reading apps. |
| **`browser`** | Default. Ordinary browser tab. |

If a browser doesn't support the chosen mode, it falls back in the order `fullscreen` → `standalone` → `minimal-ui` → `browser`.

## Service Worker

A service worker is a JavaScript file running on a separate thread that acts as a **proxy between the page, the browser, and the network**. Per MDN, it is event-driven, fully asynchronous, and has **no DOM access, no synchronous APIs, and no dynamic `import()`**. It runs only in **secure contexts (HTTPS)**; `http://localhost` is the development exception.

### Registration

```typescript
// In the page
if ('serviceWorker' in navigator) {
  window.addEventListener('load', async () => {
    try {
      const registration = await navigator.serviceWorker.register('/sw.js', {
        scope: '/',
      });
      console.log('SW registered with scope', registration.scope);
    } catch (err) {
      console.error('SW registration failed', err);
    }
  });
}
```

The **scope** defaults to the directory the SW file sits in. A worker at `/sw.js` with scope `/` controls the whole origin; a worker at `/app/sw.js` controls only `/app/*`.

> Serve `sw.js` at the **top level** if you want it to control the whole app. Bundlers that output to `/static/sw.js` will silently limit your scope.

## Lifecycle

Per web.dev's [Service Worker lifecycle](https://web.dev/articles/service-worker-lifecycle) article, a SW moves through these states:

```
register → install → waiting → activate → active → (redundant)
                        ↑                               ↓
                        └───── replaced by newer ───────┘
```

| State | What happens |
|-------|-------------|
| **Install** | Fires once per worker version. Pre-cache the app shell here. If the install promise rejects, the worker becomes `redundant`. |
| **Waiting** | A new version is installed but won't activate while the old version still controls any open pages. Ensures only one version runs at a time. |
| **Activate** | Fires when the old worker is gone. Best place to clean up old caches. Functional events (`fetch`, `push`, `sync`) are buffered until the activate promise resolves. |
| **Active** | Controlling pages and handling events. |
| **Redundant** | Discarded — install failed or a newer version replaced it. |

### `skipWaiting` and `clients.claim`

Two escape hatches from the default "safe" lifecycle:

- **`self.skipWaiting()`** — called from `install`, makes the new worker activate immediately without waiting for old pages to close. web.dev notes this is "risky if newer and older versions handle requests differently."
- **`self.clients.claim()`** — called from `activate`, makes the new worker take control of already-open pages without a reload. "This is timing sensitive" and mainly matters for the first-ever install.

```javascript
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('app-shell-v2').then(cache =>
      cache.addAll(['/', '/index.html', '/app.css', '/app.js'])
    )
  );
  self.skipWaiting(); // activate ASAP
});

self.addEventListener('activate', (event) => {
  event.waitUntil(
    (async () => {
      const keys = await caches.keys();
      await Promise.all(
        keys.filter(k => k !== 'app-shell-v2').map(k => caches.delete(k))
      );
      await self.clients.claim(); // control open pages immediately
    })()
  );
});
```

> Only use `skipWaiting` when shipping truly backward-compatible updates. Otherwise, a page might fetch assets from the new worker while running JS from the old version — classic refresh-required bug.

## Caching strategies — the Offline Cookbook

Jake Archibald's [Offline Cookbook](https://web.dev/articles/offline-cookbook) is the canonical reference. The five strategies:

| Strategy | Flow | Use for |
|----------|------|---------|
| **Cache-first** | Check cache → fall back to network | Offline-first apps; hashed static assets (JS, CSS, images) |
| **Network-first** | Try network → fall back to cache on failure | Frequently updated content (HTML pages, news feeds) |
| **Stale-while-revalidate** | Return cache immediately → fetch fresh copy in background | Content where slightly stale is OK (avatars, non-critical API responses) |
| **Network-only** | Always hit the network | Analytics, POST/PUT/DELETE — never cache non-GET |
| **Cache-only** | Always serve from cache | Pre-cached static shell assets |

```javascript
// Stale-while-revalidate
self.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return; // never cache non-GET
  event.respondWith(
    caches.open('runtime-v1').then(async (cache) => {
      const cached = await cache.match(event.request);
      const networkFetch = fetch(event.request).then((response) => {
        cache.put(event.request, response.clone());
        return response;
      });
      return cached || networkFetch;
    })
  );
});
```

> Network-first with a timeout is what you usually want for HTML: `Promise.race([fetch, timeout(3000)])` with a cache fallback.

## Background sync and push notifications

Two additional capabilities, briefly:

- **Background Sync** (`sync` event) — queues work (e.g., a failed POST) and retries it when connectivity returns. Good for "send this message even if I close the tab."
- **Push Notifications** (`push` event) — the SW receives server-pushed messages and displays them via `self.registration.showNotification()`. Requires an explicit user permission grant and, typically, a push service (FCM, Web Push protocol).

Both require a service worker and careful UX — notification-permission-prompt-on-page-load is the fastest way to burn user trust.

## Workbox — when to reach for it

[Workbox](https://developer.chrome.com/docs/workbox/) is Google's "production-ready service worker libraries and tooling." It gives you:

- Declarative **routing** and pre-built **caching strategies** (`CacheFirst`, `NetworkFirst`, `StaleWhileRevalidate`, etc.).
- **Precaching** with automatic cache versioning and cleanup.
- Plugins: expiration, broadcast updates, background sync, etc.

```javascript
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { StaleWhileRevalidate } from 'workbox-strategies';

precacheAndRoute(self.__WB_MANIFEST); // injected at build time

registerRoute(
  ({ request }) => request.destination === 'image',
  new StaleWhileRevalidate({ cacheName: 'images' })
);
```

| Use Workbox when | Write raw SW when |
|------------------|-------------------|
| You want precaching with build-time manifest generation | You have one very narrow use case (e.g., offline HTML fallback only) |
| You need several strategies per route pattern | You want zero dependencies and full control |
| You need cache expiration / versioning out of the box | You are learning the API and the indirection hurts |

## Gotchas and limitations

- **HTTPS required.** `localhost` is the only HTTP exception; everything else, including LAN IPs, needs TLS.
- **Scope is sticky.** A worker at `/app/sw.js` will never control `/`. Move the file, don't try to fight it with the `Service-Worker-Allowed` header unless you know exactly what you're doing.
- **Update cycles surprise people.** By default the new worker waits for *every* tab of the app to close. Users rarely close tabs — they reload. Either call `skipWaiting` on compatible updates, or expose an in-app "a new version is available, reload" prompt.
- **The 24-hour check.** Browsers refetch `sw.js` on navigation and at most every 24 hours, so HTTP-cache `sw.js` with `Cache-Control: no-cache` (or very short TTL) — otherwise users get pinned to a stale worker.
- **iOS Safari quirks.** PWA support is more restrictive on iOS — storage quotas are tighter, some `display` modes fall back, push only recently became available (iOS 16.4+, home-screen installed only — verify against current Safari release notes for your target iOS version).
- **Never cache non-GET.** POST/PUT/DELETE have side effects; Cache API `.match()` ignores them by default, but your `fetch` handler might not.

---

[← Previous: Web Performance](06-web-performance.md) | [Next: SPA vs SSR vs SSG →](08-spa-ssr-ssg.md) | [Back to index](README.md)
