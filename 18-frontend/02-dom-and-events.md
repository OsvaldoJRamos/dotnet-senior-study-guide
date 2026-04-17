# DOM and Events

If you come from .NET, the closest mental model for the **DOM** is an in-memory object tree that the browser re-renders whenever you mutate it. Events are how the tree tells you something happened — the catch is they travel through the tree in a specific order, and several APIs exist to intercept or ignore them.

## The DOM Tree and Element References

Every element, text node, and comment in the page is a node in the DOM tree. Two APIs cover 95% of the "get me that element" work:

```ts
// By CSS selector — returns the first match or null
const submit = document.querySelector<HTMLButtonElement>('form button[type=submit]');

// By id — faster, but only by id
const nav = document.getElementById('nav');

// Multiple matches — returns a static NodeList
const items = document.querySelectorAll<HTMLLIElement>('li.item');

// "Live" collection — updates when the DOM changes
const forms = document.forms;
```

`querySelector` / `querySelectorAll` return **static** snapshots for `NodeList` results; `getElementsByClassName` and `document.forms` / `document.images` return **live** collections that re-evaluate as the DOM changes. If you iterate a live collection and mutate the DOM in the loop, you can walk off the end.

## Event Phases: Capture, Target, Bubble

An event dispatched at an element travels through three phases, defined on `Event.eventPhase` (MDN):

| Phase | Constant | What happens |
|-------|----------|--------------|
| **Capturing** | `CAPTURING_PHASE` (1) | Event travels **down** from the window through ancestors toward the target |
| **Target** | `AT_TARGET` (2) | Event is at the element it was dispatched on |
| **Bubbling** | `BUBBLING_PHASE` (3) | Event travels **up** from the target back through ancestors |

By default, listeners registered with `addEventListener(type, fn)` fire during the **bubble** phase. Pass `{ capture: true }` to fire during capture instead.

```ts
document.body.addEventListener('click', () => console.log('bubble on body'));
document.body.addEventListener('click', () => console.log('capture on body'), { capture: true });
```

Not every event bubbles (e.g., `focus`, `blur` do not; use `focusin` / `focusout` if you need bubbling equivalents).

## `stopPropagation` vs `stopImmediatePropagation` vs `preventDefault`

Three methods, three different jobs. Senior interview bait.

| Method | Stops further listeners on **other** elements? | Stops further listeners on **same** element? | Cancels default action? |
|--------|-----------------------------------------------|----------------------------------------------|-------------------------|
| `preventDefault()` | No | No | Yes (if cancelable) |
| `stopPropagation()` | Yes | No | No |
| `stopImmediatePropagation()` | Yes | Yes | No |

MDN:
- `preventDefault()` "cancels the event (if it is cancelable)" — for example, prevents form submission or link navigation.
- `stopPropagation()` "stops the propagation of events further along in the DOM."
- `stopImmediatePropagation()` "prevents all other listeners from being called for this particular event."

Classic pitfall: calling `preventDefault()` does **not** stop propagation, and calling `stopPropagation()` does **not** cancel the default action. They are independent.

```ts
form.addEventListener('submit', (e) => {
  e.preventDefault();      // don't actually submit the form
  // the submit event still bubbles up
});
```

## Passive Listeners

`{ passive: true }` tells the browser: "I will not call `preventDefault()` from this listener, so start the default action immediately."

Why it matters: on a non-passive `touchstart` / `touchmove` listener, the browser must wait for your handler to finish before it knows whether to scroll. That single listener can stall scrolling for milliseconds per event. Passive listeners unblock scroll.

MDN notes that modern browsers (outside Safari) default `passive` to `true` for `wheel`, `mousewheel`, `touchstart`, and `touchmove` listeners added on `Window`, `Document`, or `Document.body`. Everywhere else the default is `false`.

```ts
window.addEventListener('scroll', onScroll, { passive: true });
```

If you call `preventDefault()` inside a passive listener, the call is ignored, the default action proceeds, and the browser may log a console warning (MDN).

## Event Delegation

Instead of attaching N listeners to N children, attach one listener to the common ancestor and use `event.target` to figure out which child fired. Works because events bubble.

```ts
// One listener for an entire list, even if rows are added later.
list.addEventListener('click', (e) => {
  const row = (e.target as HTMLElement).closest('li[data-id]');
  if (!row) return;
  const id = row.dataset.id;
  removeItem(id!);
});
```

Good for: long lists, dynamically added content, reducing memory. Bad for: events that don't bubble (`focus`, `blur`, `load` on descendants, `mouseenter` / `mouseleave`).

## `CustomEvent`

Dispatching your own events is a lightweight way to let components communicate without coupling them to each other. `CustomEvent` carries arbitrary payload in its `detail` property (MDN).

```ts
// Fire
el.dispatchEvent(
  new CustomEvent<{ orderId: string }>('order:placed', {
    detail: { orderId: 'O-123' },
    bubbles: true,
    cancelable: true,
  }),
);

// Handle
document.addEventListener('order:placed', (e) => {
  const ce = e as CustomEvent<{ orderId: string }>;
  console.log(ce.detail.orderId);
});
```

> Prefer `CustomEvent` over a home-grown event bus when the communication is already tree-shaped (parent-child or ancestor-descendant). Use a framework state/store for cross-tree communication.

## Memory Leaks from Detached DOM and Listeners

A **detached DOM** leak happens when JavaScript still holds a reference to an element that is no longer in the document. The element, its subtree, and any closures referenced by its listeners stay in memory.

Typical sources:

```ts
// Holding a cached reference to a removed element
let panel = document.getElementById('panel');
panel!.remove();     // removed from the tree
// panel still points to it → the whole subtree is retained

// Listener on a long-lived target that closes over a removed element
function onResize() { doSomething(panel); }
window.addEventListener('resize', onResize);
// Until removeEventListener runs, the closure keeps panel alive
```

Fixes:

- Null out cached references when you're done with them.
- Always pair `addEventListener` with `removeEventListener`, or register with `{ signal }` and call `controller.abort()` to remove everything at once.

```ts
const controller = new AbortController();
window.addEventListener('resize', onResize, { signal: controller.signal });
window.addEventListener('scroll', onScroll, { signal: controller.signal, passive: true });
// Later — one call cleans up both:
controller.abort();
```

In Chrome DevTools, a **Memory** panel heap snapshot with the "Detached" filter surfaces retained detached nodes quickly.

## `MutationObserver`

Observes changes to the DOM tree — added/removed children, attribute changes, character data changes. Callbacks are **asynchronous** and batch bursts of DOM edits rather than firing on every single mutation (per the WHATWG DOM spec, which queues a microtask to deliver records).

```ts
const mo = new MutationObserver((records) => {
  for (const r of records) {
    if (r.type === 'childList') console.log('children changed on', r.target);
    if (r.type === 'attributes') console.log('attr', r.attributeName, 'changed');
  }
});

mo.observe(document.getElementById('app')!, {
  childList: true,
  attributes: true,
  subtree: true, // watch the entire descendant tree
});

// Later
mo.disconnect();
```

Use for: reacting to DOM changes you don't control (third-party widgets, integrating with legacy code). Don't use it as a state-management tool when a framework already owns the tree.

## `IntersectionObserver`

Fires when a target element's visibility relative to the viewport (or a specified root) crosses a **threshold**. Designed explicitly as a performant replacement for scroll listeners plus manual `getBoundingClientRect` math (MDN).

```ts
const io = new IntersectionObserver(
  (entries) => {
    for (const entry of entries) {
      if (entry.isIntersecting) {
        loadImage(entry.target as HTMLImageElement);
        io.unobserve(entry.target);
      }
    }
  },
  {
    root: null,               // null = viewport
    rootMargin: '200px 0px',  // trigger 200px before the element enters the viewport
    threshold: 0,             // fire as soon as any part enters
  },
);

document.querySelectorAll<HTMLImageElement>('img.lazy').forEach((img) => io.observe(img));
```

Typical uses: lazy-loading images, infinite scroll, firing analytics when a section becomes visible, pausing video when off-screen.

`threshold` can be a number or an array (e.g., `[0, 0.25, 0.5, 0.75, 1]`) if you want multiple fire points.

## `ResizeObserver`

Fires when an **element's** content box or border box changes size — not just the viewport. That's the key difference from `window.resize`, which only tells you about the viewport (MDN).

```ts
const ro = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const size = entry.contentBoxSize[0]; // [0] = the first (usually only) fragment
    console.log('inline:', size.inlineSize, 'block:', size.blockSize);
  }
});

ro.observe(document.getElementById('chart')!);
```

Each entry exposes `contentBoxSize`, `borderBoxSize`, and `contentRect`. MDN: the content box is the area where content sits (excluding padding and border); the border box encompasses content + padding + border.

Use for: charts that must re-render when their container resizes, responsive component-level logic that can't rely on CSS media queries, or detecting when inline content changes the size of its parent.

> All three observers (`Mutation`, `Intersection`, `Resize`) deliver callbacks asynchronously, so using them inside a render path won't force layout synchronously — a nice property for performance-sensitive code.

## Quick Reference: Which Observer?

| You need to know when... | Use |
|--------------------------|-----|
| DOM nodes / attributes change | `MutationObserver` |
| An element enters or leaves the viewport | `IntersectionObserver` |
| An element's size changes | `ResizeObserver` |
| The window itself resizes | `window.addEventListener('resize', ...)` |

---

[← Previous: Browser Rendering](01-browser-rendering.md) | [Next: CDN →](03-cdn.md) | [Back to index](README.md)
