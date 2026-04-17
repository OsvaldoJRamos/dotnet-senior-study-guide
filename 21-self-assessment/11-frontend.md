# Frontend

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the difference between a Promise and an Observable? When would you use each?

<details>
<summary>Reveal answer</summary>

| Aspect | Promise | Observable |
|--------|---------|------------|
| Values | Single value | Stream of 0..N values over time |
| Execution | Eager (starts immediately) | Lazy (starts on `subscribe()`) |
| Cancellation | Not natively cancellable | Cancellable via `unsubscribe()` |
| Operators | `.then()` / `.catch()` | Rich operator library (RxJS) |
| Multicasting | Always shared | Unicast by default, multicast with `share()` |

**Use Promises** for one-shot async operations where you don't need cancellation (e.g., a simple fetch during app init). **Use Observables** for streams, repeated events, or when you need cancellation and composition (HTTP requests in Angular, form value changes, WebSocket messages).

Angular's `HttpClient` returns Observables, which lets you use operators like `retry`, `switchMap`, and cancel in-flight requests when a component is destroyed.

Deep dive: [Promises vs Observables](../18-frontend/10-promises-vs-observables.md)

</details>

---

### 2. What does `switchMap` do, and why is it the go-to operator for type-ahead search?

<details>
<summary>Reveal answer</summary>

`switchMap` subscribes to an inner Observable for each emission of the outer Observable, but **cancels the previous inner subscription** when a new value arrives.

For a type-ahead search:

```typescript
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.searchService.search(term))
).subscribe(results => this.results = results);
```

If the user types "ang" and then "angular", `switchMap` cancels the HTTP request for "ang" as soon as "angular" arrives. This prevents **race conditions** where a slower first response could overwrite a faster second response.

Other mapping operators for comparison:
- `mergeMap` -- keeps all inner subscriptions alive (parallel).
- `concatMap` -- queues inner subscriptions (sequential).
- `exhaustMap` -- ignores new emissions while an inner subscription is active (good for login buttons).

Deep dive: [Promises vs Observables](../18-frontend/10-promises-vs-observables.md)

</details>

---

### 3. What is the `async` pipe, and why should you prefer it over manual `subscribe()`?

<details>
<summary>Reveal answer</summary>

The `async` pipe subscribes to an Observable (or Promise) in the template and **automatically unsubscribes** when the component is destroyed.

```html
<div *ngFor="let item of items$ | async">{{ item.name }}</div>
```

**Advantages over manual subscribe**:
1. **No memory leaks** -- unsubscription is automatic.
2. **Less boilerplate** -- no `ngOnDestroy`, no `Subscription` tracking.
3. **Works with OnPush** -- the pipe triggers `markForCheck()`, so change detection fires correctly.
4. **Cleaner code** -- no intermediate properties to store the unwrapped value.

**When manual subscribe is still needed**: when you need to perform side effects (navigation, toasts), or complex orchestration that doesn't map cleanly to template binding.

Deep dive: [Promises vs Observables](../18-frontend/10-promises-vs-observables.md)

</details>

---

### 4. How does OnPush change detection work, and what triggers a re-render?

<details>
<summary>Reveal answer</summary>

With `ChangeDetectionStrategy.OnPush`, Angular **skips** the component's subtree during change detection unless one of these triggers occurs:

1. **@Input reference changes** -- the parent passes a new object reference (not mutation).
2. **Event handler fires** -- a DOM event bound in the template (click, keyup, etc.).
3. **Async pipe emits** -- the `async` pipe calls `markForCheck()` internally.
4. **Manual trigger** -- you call `ChangeDetectorRef.markForCheck()` or `detectChanges()`.

**Immutability requirement**: if you mutate an object and pass it as `@Input`, OnPush won't detect the change because the reference is the same. You must create a new object:

```typescript
// Won't trigger OnPush
this.user.name = 'New Name';

// Will trigger OnPush
this.user = { ...this.user, name: 'New Name' };
```

OnPush is a major performance win for large component trees.

Deep dive: [Angular Performance](../18-frontend/12-angular-performance.md)

</details>

---

### 5. What is `trackBy` in `*ngFor`, and why does it matter for performance?

<details>
<summary>Reveal answer</summary>

By default, `*ngFor` tracks items by **object identity** (reference). If you replace the array (e.g., after an API call), Angular destroys and recreates **all** DOM elements, even if the data hasn't changed.

`trackBy` tells Angular how to identify each item so it can **reuse existing DOM nodes**:

```typescript
trackById(index: number, item: Product): number {
  return item.id;
}
```

```html
<div *ngFor="let item of items; trackBy: trackById">{{ item.name }}</div>
```

**Impact**: without `trackBy`, re-fetching a list of 500 items recreates 500 DOM elements. With `trackBy`, Angular only updates the elements whose data actually changed. This dramatically reduces DOM manipulation and improves rendering performance.

Deep dive: [Angular Performance](../18-frontend/12-angular-performance.md)

</details>

---

### 6. How do you set up lazy loading for Angular modules, and why does it improve performance?

<details>
<summary>Reveal answer</summary>

Lazy loading defers downloading a module's JavaScript until the user navigates to its route:

```typescript
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module')
      .then(m => m.AdminModule)
  }
];
```

With standalone components (Angular 14+):

```typescript
{
  path: 'admin',
  loadComponent: () => import('./admin/admin.component')
    .then(c => c.AdminComponent)
}
```

**Why it matters**:
- **Smaller initial bundle** -- users download only what they need for the first page.
- **Faster Time to Interactive** -- less JavaScript to parse and execute on startup.
- **Separate chunks** -- the build system creates separate JS files per lazy module.

**Preloading strategies** like `PreloadAllModules` can load lazy modules in the background after the app bootstraps, giving you the best of both worlds.

Deep dive: [Angular Performance](../18-frontend/12-angular-performance.md)

</details>

---

### 7. What is virtual scrolling, and when should you use it?

<details>
<summary>Reveal answer</summary>

Virtual scrolling renders **only the DOM elements visible in the viewport**, plus a small buffer. As the user scrolls, elements are recycled -- old ones are destroyed and new ones are created.

Angular CDK provides `cdk-virtual-scroll-viewport`:

```html
<cdk-virtual-scroll-viewport itemSize="48" class="list-viewport">
  <div *cdkVirtualFor="let item of items">{{ item.name }}</div>
</cdk-virtual-scroll-viewport>
```

**When to use**: lists with **hundreds or thousands** of items where rendering all DOM nodes would cause jank (slow scrolling, high memory usage). A list of 10,000 items without virtual scrolling creates 10,000 DOM nodes; with virtual scrolling, only ~20-30 are in the DOM at any time.

**Trade-offs**: items must have a known (or estimable) height, and features like "find in page" (Ctrl+F) won't find off-screen items since they don't exist in the DOM.

Deep dive: [Angular Performance](../18-frontend/12-angular-performance.md)

</details>

---

### 8. What are the main challenges when using SignalR with Angular, and how do you handle them?

<details>
<summary>Reveal answer</summary>

**Key challenges and solutions**:

1. **Zone.js integration** -- SignalR callbacks run outside Angular's zone, so change detection doesn't fire. Fix: inject `NgZone` and wrap callbacks:

```typescript
this.hubConnection.on('ReceiveMessage', (msg) => {
  this.ngZone.run(() => {
    this.messages.push(msg);
  });
});
```

In **zoneless Angular (17+)** with signals, prefer updating signals inside the callback (no `NgZone.run()` needed); zone-based change detection is being phased out.

2. **Reconnection** -- connections drop due to network issues. Use the built-in reconnection with `withAutomaticReconnect()`:

```typescript
this.hubConnection = new signalR.HubConnectionBuilder()
  .withUrl('/chatHub')
  .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
  .build();
```

3. **Lifecycle management** -- start the connection in `ngOnInit` or a service constructor, and stop it in `ngOnDestroy` (or when the service is destroyed) to avoid memory leaks.

4. **Authentication** -- pass the JWT token via the `accessTokenFactory` option so the WebSocket connection is authenticated.

5. **Scalability** -- in multi-server deployments, use a **backplane** (Redis, Azure SignalR Service) so messages reach all connected clients regardless of which server they're connected to.

Deep dive: [SignalR with Angular](../18-frontend/13-signalr-with-angular.md)

</details>

---

### 9. What is the difference between a cold and a hot Observable? Give an example of each.

<details>
<summary>Reveal answer</summary>

A **cold Observable** starts a **new, independent execution for every subscriber**. Each subscriber sees its own values from the beginning.

A **hot Observable** shares a **single execution** across all subscribers. Values are produced whether or not anyone is subscribed, and late subscribers miss earlier emissions.

**Cold examples**: `of(1, 2, 3)`, `interval(1000)`, `this.http.get('/api/x')`. Two subscribers to the same `http.get` trigger **two HTTP requests**.

**Hot examples**: `fromEvent(button, 'click')`, any `Subject` / `BehaviorSubject`, or any Observable piped through `share()` / `shareReplay()`. The canonical fix when two components accidentally trigger the same request:

```typescript
config$ = this.http.get<Config>('/api/config').pipe(shareReplay(1));
```

Now both subscribers share one execution, and the last value is replayed to late subscribers.

Deep dive: [RxJS Fundamentals](../18-frontend/11-rxjs-fundamentals.md)

</details>

---

### 10. Which RxJS operators would you use to build an autocomplete search input? Why?

<details>
<summary>Reveal answer</summary>

The canonical pipeline:

```typescript
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term => this.api.search(term)),
  takeUntilDestroyed(this.destroyRef),
).subscribe(results => this.results = results);
```

Why each operator:

- **`debounceTime(300)`** — waits for the user to stop typing before hitting the API. Avoids a request on every keystroke.
- **`distinctUntilChanged()`** — skips when the value hasn't changed (e.g., the user hit a modifier key).
- **`switchMap`** — the critical one. When a new term arrives while the previous request is still in flight, `switchMap` **cancels** the previous request and subscribes to the new one. This prevents race conditions where a slow response for "ang" could overwrite a faster response for "angular".
- **`takeUntilDestroyed`** — ties the subscription lifetime to the component so it cleans up on destroy.

`mergeMap` would be wrong here — it would keep all in-flight requests running, letting stale responses win. `concatMap` would serialize, forcing the user to wait for every obsolete request to finish.

Deep dive: [RxJS Fundamentals](../18-frontend/11-rxjs-fundamentals.md)

</details>

---

### 11. How do you prevent memory leaks from RxJS subscriptions in Angular components?

<details>
<summary>Reveal answer</summary>

A subscription that is never unsubscribed keeps the producer alive and its callbacks running — even after the component is destroyed. Preferred patterns, from best to legacy:

1. **`async` pipe in the template** — Angular subscribes on view init and unsubscribes on destroy automatically. No cleanup code at all.

   ```html
   <div *ngFor="let user of users$ | async">{{ user.name }}</div>
   ```

2. **`takeUntilDestroyed()`** (Angular 16+) — for manual `.subscribe()` calls. It completes the Observable when the component's `DestroyRef` fires. Called without arguments, it needs to run in an injection context (field initializer or constructor); otherwise pass an explicit `DestroyRef`.

   ```typescript
   private destroyRef = inject(DestroyRef);
   ngOnInit() {
     this.service.stream$.pipe(takeUntilDestroyed(this.destroyRef)).subscribe(...);
   }
   ```

3. **`takeUntil(destroy$)`** — legacy pattern using a manual `Subject` that emits in `ngOnDestroy`. Still works, just verbose.

4. **Manual `subscription.unsubscribe()` in `ngOnDestroy`** — last resort; easy to forget, and noisy when you have several subscriptions.

The rule of thumb: prefer `async` pipe; if you must subscribe in code, always pair it with `takeUntilDestroyed`.

Deep dive: [RxJS Fundamentals](../18-frontend/11-rxjs-fundamentals.md)

</details>

---

### 12. Explain the Critical Rendering Path. What is the difference between reflow, repaint, and composite?

<details>
<summary>Reveal answer</summary>

Pixel pipeline: **JavaScript → Style → Layout → Paint → Composite**.

- **Reflow (Layout)**: geometry changed — width, height, top, left, font-size. Re-runs layout; expensive.
- **Repaint**: visual-only change that does not affect layout — `color`, `background-image`, `box-shadow`. Skips layout; still costs main-thread work.
- **Composite only**: `transform` and `opacity` on a promoted layer. Handled by the compositor thread — cheapest and does not block the main thread.

Rule of thumb for animations: stick to `transform` and `opacity`. `will-change` is a last-resort hint to promote layers.

Deep dive: [Browser Rendering](../18-frontend/01-browser-rendering.md)

</details>

---

### 13. What are the event phases in the DOM? When would you use event delegation?

<details>
<summary>Reveal answer</summary>

Three phases: **capturing** (root → target), **target**, **bubbling** (target → root). `addEventListener` uses bubbling by default; pass `{ capture: true }` for the capturing phase.

**Event delegation**: attach one listener on a common ancestor and inspect `event.target`. Use when you have many similar children, children are added/removed dynamically, or you want to avoid binding thousands of listeners.

Use `{ passive: true }` for `scroll`/`touchstart`/`wheel` — lets the browser scroll on the compositor thread. In a passive listener, `preventDefault()` is a no-op.

Deep dive: [DOM and Events](../18-frontend/02-dom-and-events.md)

</details>

---

### 14. What is the difference between `Cache-Control: max-age` and `s-maxage`? What does `stale-while-revalidate` do?

<details>
<summary>Reveal answer</summary>

- `max-age=N` applies to **any cache** (browser + CDN).
- `s-maxage=N` applies to **shared caches only** (CDN / proxy). Lets you tell the CDN "hold for 1 hour" while the browser revalidates more often.
- `stale-while-revalidate=N` (RFC 5861): the cache serves the stale response for N seconds after expiry while it revalidates in the background. Users see instant responses; the cache catches up asynchronously.
- `stale-if-error=N`: serve stale on 5xx errors from the origin — keeps the site up during outages.
- `immutable`: tells the browser not to revalidate even on reload (use with hashed asset filenames).

Deep dive: [CDN](../18-frontend/03-cdn.md)

</details>

---

### 15. Why should you never call `postMessage` with target `'*'`? `X-Frame-Options` vs CSP `frame-ancestors`?

<details>
<summary>Reveal answer</summary>

**`postMessage(data, '*')`** sends the message to any origin currently in the target window. A malicious page could navigate the frame and receive your data. Always specify the target origin and verify `event.origin` in the receiver.

**`X-Frame-Options`** (legacy): `DENY` or `SAMEORIGIN` only. `ALLOW-FROM` is obsolete.

**CSP `frame-ancestors`** (modern): expressive allowlist with wildcards, `'none'`, `'self'`. Preferred. Ship both for defense in depth.

The `sandbox` attribute strips iframe privileges (scripts, forms, same-origin, top-nav). Opt in with `allow-*` tokens only as needed.

Deep dive: [iframes and Embedding](../18-frontend/04-iframes-and-embedding.md)

</details>

---

### 16. When would you choose cookies over localStorage for a JWT? What flags are required?

<details>
<summary>Reveal answer</summary>

Use a cookie to prevent XSS from stealing the token. JavaScript cannot read an `HttpOnly` cookie.

Required flags for session cookies:
- **`HttpOnly`** — blocks `document.cookie` access.
- **`Secure`** — only sent over HTTPS (except localhost).
- **`SameSite=Lax`** (default) or **`Strict`** — blocks CSRF on cross-site POSTs.
- **`SameSite=None`** only when cross-site sending is required, and then `Secure` is mandatory.
- `__Host-` prefix: forces `Secure`, no `Domain`, `Path=/` — hardens against subdomain attacks.

Trade-off: cookies are sent on every same-site request. localStorage is fine for non-sensitive UI state.

Deep dive: [Web Storage](../18-frontend/05-web-storage.md)

</details>

---

### 17. What are the three Core Web Vitals and their "good" thresholds? Why did INP replace FID?

<details>
<summary>Reveal answer</summary>

| Metric | Measures | Good | Poor |
|--------|----------|------|------|
| **LCP** | Largest Contentful Paint — loading | ≤ 2.5 s | > 4.0 s |
| **INP** | Interaction to Next Paint — responsiveness | ≤ 200 ms | > 500 ms |
| **CLS** | Cumulative Layout Shift — visual stability | ≤ 0.1 | > 0.25 |

Evaluated at the **75th percentile** of visits.

**INP replaced FID** on 12 March 2024 (FID support ended 9 September 2024). FID only measured the *first* input delay. INP samples **every** interaction and reports the worst — a far better proxy for real responsiveness.

Deep dive: [Web Performance](../18-frontend/06-web-performance.md)

</details>

---

### 18. Walk through the Service Worker lifecycle. What does `skipWaiting()` do and why is it risky?

<details>
<summary>Reveal answer</summary>

**install → waiting → activate → active → redundant**.

- **install**: pre-cache assets here.
- **waiting**: a new SW is installed but the old one still controls open tabs.
- **activate**: the new SW takes over; clean up old caches.
- **redundant**: replaced or discarded.

**`skipWaiting()`** forces the new SW to activate immediately while tabs are open. Risky because a running page may have already received responses from the old SW — mixing versions in a single session leads to inconsistent behavior. Safe only when the two versions are interchangeable.

**`clients.claim()`** takes over existing uncontrolled tabs on activate. Pair with `skipWaiting` cautiously.

Deep dive: [PWA and Service Workers](../18-frontend/07-pwa-service-workers.md)

</details>

---

### 19. Trade-offs between SPA, SSR, SSG, and ISR. What is hydration and why is it expensive?

<details>
<summary>Reveal answer</summary>

| Strategy | TTFB | LCP | SEO | Best for |
|----------|------|-----|-----|----------|
| **SPA (CSR)** | Fast | Slow | Weak w/o pre-render | Authed dashboards |
| **SSR** | Slow | Fast | Strong | Dynamic content + SEO |
| **SSG** | Fast | Fast | Strong | Docs, marketing, blogs |
| **ISR** | Fast | Fast | Strong | Mostly-static with occasional updates |

**Hydration**: the server ships HTML, then the client ships the same JS bundle and re-runs the framework to attach listeners and reconcile the DOM. Expensive because it downloads the full bundle, duplicates CPU work, and blocks interactivity until complete (hurts INP).

Partial hydration (Astro islands), resumability (Qwik), and React Server Components reduce or eliminate this cost.

Deep dive: [SPA vs SSR vs SSG](../18-frontend/08-spa-ssr-ssg.md)

</details>

---

### 20. What are the four WCAG principles (POUR)? What is the "first rule of ARIA"?

<details>
<summary>Reveal answer</summary>

**POUR**:
- **Perceivable** — alt text, captions, contrast ≥ 4.5:1 body / 3:1 large.
- **Operable** — keyboard-reachable, no seizure triggers, enough time.
- **Understandable** — predictable navigation, clear errors.
- **Robust** — works with current and future user agents, including assistive tech.

Conformance levels: **A / AA / AAA**. AA is the typical legal target (EAA, ADA, Section 508, EN 301 549).

**First rule of ARIA**: *"If you can use a native HTML element or attribute with the semantics and behavior you require already built in, instead of re-purposing an element and adding an ARIA role, state or property to make it accessible, then do so."*

A `<button>` gives keyboard focus, Enter/Space activation, and screen-reader semantics for free.

Deep dive: [Accessibility](../18-frontend/09-accessibility.md)

</details>

---
