# SPA vs SSR vs SSG vs ISR

Four rendering strategies for the web. The question is where and when HTML is produced: on the client after JS boots (SPA), on the server per request (SSR), at build time (SSG), or at build time with background revalidation (ISR). The right answer depends on SEO needs, freshness requirements, server cost, and TTFB vs interaction trade-offs.

## The four strategies

| Strategy | HTML produced | Freshness | Server cost | Typical stack |
|----------|---------------|-----------|-------------|---------------|
| **SPA (CSR)** | In the browser after JS bundle loads | Always live | Static host only | Angular, React, Vue |
| **SSR** | On the server, per request | Always live | Pay per request | Angular SSR, Next.js, Nuxt, ASP.NET Core MVC / Razor Pages |
| **SSG** | At build time | Stale until rebuild | Static host only | Next.js `generateStaticParams`, Astro, Hugo, 11ty |
| **ISR** | At build time, regenerated in background on revalidate | Eventually fresh | Hybrid (static + occasional render) | Next.js (native), Astro adapters |

## SPA (Client-Side Rendering)

The server returns a nearly empty HTML shell plus a JS bundle. The browser downloads the bundle, boots the framework, then renders the page and fetches data.

```html
<!-- Everything interesting happens after app.js loads -->
<body>
  <div id="app"></div>
  <script src="/app.js"></script>
</body>
```

**Pros**
- Cheap to host: static files on a CDN
- Fast navigation after the first load (no server round-trip for new routes)
- Clean separation between backend API and frontend app

**Cons**
- Slow first paint: blank white screen until JS parses, executes, and fetches data
- Bad for SEO without extra work (crawlers see an empty shell — Googlebot renders JS, most others still struggle)
- Large JS bundles hurt low-end devices and slow networks
- Requires a fallback (prerender, SSR, or meta tags) for social-media previews (OG/Twitter cards don't run JS)

**When to use:** authenticated dashboards, internal tools, back-office apps. Anything behind a login where SEO doesn't matter and the first paint is acceptable behind a spinner.

## SSR (Server-Side Rendering)

The server renders the component tree to HTML per request and streams it to the browser. The browser shows content immediately, then **hydrates** to make it interactive.

```ts
// Angular SSR: bootstrap the same app on the server per request
import { bootstrapApplication } from '@angular/platform-browser';
import { provideClientHydration } from '@angular/platform-browser';

bootstrapApplication(AppComponent, {
  providers: [provideClientHydration()]
});
```

**Pros**
- Fast FCP/LCP: the browser has real HTML before JS runs
- Good SEO and social previews out of the box
- Works on slow networks and low-end devices (the first paint doesn't wait for the bundle)

**Cons**
- TTFB is coupled to server work — a slow DB query becomes a slow first byte
- Server cost scales with traffic (every request renders)
- Hydration cost: the same component tree runs twice (server + client), doubling the CPU work on the client
- State serialization is a common trap — data fetched on the server must be transferred to the client to avoid a double-fetch

> **Angular SSR + `TransferState` / HTTP transfer cache.** Angular's `HttpClient` automatically caches outgoing requests during server render, serializes them into the HTML payload, and reuses them on the client so you don't re-fetch after hydration. Configure via `withHttpTransferCacheOptions`. Source: angular.dev/guide/ssr.

**When to use:** marketing pages, e-commerce product pages, blogs with comments, anything where SEO + fresh-per-request data matter.

## SSG (Static Site Generation)

HTML is rendered **once** at build time for every known route and uploaded as plain files to a CDN.

```ts
// Next.js: generate static params at build
export async function generateStaticParams() {
  const posts = await fetchAllPosts();
  return posts.map(p => ({ id: p.id }));
}
```

**Pros**
- Best possible performance: just files on a CDN, no server render on the critical path
- Cheapest to host (no compute per request)
- No origin to DDoS

**Cons**
- Content is stale until the next build — a typo fix means a full redeploy
- Build times explode with large content sets (10k+ pages can take 10+ min)
- Doesn't fit per-user content (pricing by region, logged-in state)

**When to use:** docs sites, marketing sites, blogs, changelog pages — anything where content changes in minutes-to-hours, not seconds.

## ISR (Incremental Static Regeneration)

Next.js's hybrid: pages are pre-rendered at build but the cache is invalidated on a timer (`revalidate = 60`) or on-demand (`revalidatePath`, `revalidateTag`). A request after expiry gets the **stale** cached page instantly while Next regenerates in the background — classic stale-while-revalidate.

```tsx
// app/blog/[id]/page.tsx
export const revalidate = 60; // at most once every 60s

export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/blog').then(r => r.json());
  return posts.map((p: { id: string }) => ({ id: p.id }));
}
```

On-demand invalidation from a Server Action:

```ts
'use server';
import { revalidatePath } from 'next/cache';

export async function publishPost() {
  // ... write to DB
  revalidatePath('/blog');
}
```

**Gotchas from the Next.js docs**
- Not supported with `output: 'export'` (static export)
- The default file-system cache is **per instance** — on-demand revalidation only invalidates the instance that received the call. Use a shared cache handler for multi-instance deploys
- Background regeneration runs on the instance that received the triggering request, so it counts as compute on per-request billing platforms
- If any `fetch` in the route uses `no-store` or `revalidate: 0`, the route falls back to fully dynamic rendering

**When to use:** content that updates on human time scales (minutes-to-hours) but needs to be fresh after editorial changes — news sites, large catalogs, marketing pages with CMS-driven content.

## Hydration — the hidden cost of SSR

Hydration is the process where the client-side framework attaches event listeners and state to the server-rendered HTML without re-rendering it. The tree is rendered **twice**: once on the server (producing HTML) and once on the client (producing the component tree that matches the HTML).

**Hydration costs:**
- Downloading the full JS bundle
- Parsing + executing the framework
- Walking the DOM to attach listeners
- Re-running all component render functions to rebuild the virtual tree

For a large app this can add hundreds of milliseconds of blocked main thread — bad for INP and first interaction.

### Strategies to reduce hydration cost

| Strategy | How it works | Available in |
|----------|--------------|--------------|
| **Partial hydration / Islands** | Only interactive components hydrate; static parts ship zero JS | Astro, Fresh, Marko |
| **Streaming SSR** | Server flushes HTML in chunks as it's ready (uses `<Suspense>`) | React 18+, Next.js App Router |
| **React Server Components (RSC)** | Components run on the server and send rendered output (not code) to the client; stable in React 19 | React 19, Next.js App Router |
| **Incremental hydration** | Dehydrated `@defer` blocks hydrate on idle/viewport/interaction/hover triggers | Angular (via `@defer` with hydrate triggers) |
| **Resumability** | No hydration at all — the framework serializes execution state in HTML and resumes on interaction | Qwik |

> **RSC vs SSR.** Server Components aren't a form of SSR — their code stays on the server and never ships to the client. SSR pre-renders HTML but still ships the component code for hydration. You can combine both: SSR the initial HTML, with RSC for parts that don't need client-side interactivity. Source: react.dev/reference/rsc/server-components.

## Trade-off table

| Metric | SPA | SSR | SSG | ISR |
|--------|-----|-----|-----|-----|
| **TTFB** | Fastest (static shell) | Slower (server work per request) | Fastest (CDN) | Fastest when cached, SSR-speed on miss |
| **FCP / LCP** | Slow (waits for JS + data) | Fast | Fastest | Fastest (served stale) |
| **INP (interactivity)** | Fast after boot, slow to boot | Hydration cost on first load | Same as SSR if hydrated | Same as SSR if hydrated |
| **SEO** | Weak without prerender | Strong | Strong | Strong |
| **Server cost** | Near zero | Scales with traffic | Near zero | Near zero + occasional regenerate |
| **Freshness** | Always live (client fetches) | Always live | Stale until rebuild | Stale up to `revalidate` window |
| **Developer cost** | Low | Medium (two runtimes) | Low | Medium (cache mental model) |

## Decision guide

- **Logged-in dashboard, SEO irrelevant, fine with a spinner** → SPA
- **Marketing / e-commerce / blog, content changes all the time** → SSR (or ISR with aggressive revalidation)
- **Docs site, changelog, blog posts that rarely change** → SSG
- **CMS-driven site, editorial changes must go live in minutes without redeploying** → ISR
- **Large JS-heavy app where INP matters more than FCP** → Islands (Astro) or RSC (Next 14+)

> Rendering strategy isn't global. Modern frameworks let you pick per-route: static marketing pages, SSR product pages, CSR account dashboard — in the same app.

---

[← Previous: PWA and Service Workers](07-pwa-service-workers.md) | [Next: Accessibility →](09-accessibility.md) | [Back to index](README.md)
