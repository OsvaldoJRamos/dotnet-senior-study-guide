# Browser Rendering (Critical Rendering Path)

The **Critical Rendering Path** is the sequence the browser follows to turn HTML, CSS, and JavaScript into pixels on the screen. Understanding it is the difference between "my animation is janky, no idea why" and "I moved that property to the compositor and shipped 60fps."

## The Pixel Pipeline

Every visual change flows through some portion of this pipeline. The stages come from web.dev's "Rendering performance" article (the terms are **JavaScript → Style → Layout → Paint → Composite**):

| Step | What the browser does |
|------|-----------------------|
| **JavaScript** | Runs code that triggers visual work (DOM updates, animation drivers, etc.) |
| **Style** | Matches CSS selectors to elements, recomputes computed styles |
| **Layout** | Calculates geometry: width, height, and position of every element |
| **Paint** | Fills pixels — text, colors, images, borders, shadows — across one or more layers |
| **Composite** | Assembles the painted layers into the final on-screen image in the correct order |

Before any of this can happen on first load, the browser parses HTML into the **DOM**, parses CSS into the **CSSOM**, and combines them into the **render tree** (only visible nodes — elements with `display: none` are excluded). CSS is **render-blocking**: the CSSOM must be fully parsed before the render tree can be built (MDN, "Critical rendering path").

## Reflow vs Repaint vs Composite

Not every change costs the same. Which CSS properties you touch decides how much of the pipeline re-runs.

| Change type | Pipeline path | Trigger examples | Cost |
|-------------|---------------|------------------|------|
| **Layout (reflow)** | Style → Layout → Paint → Composite | `width`, `height`, `top`, `left`, `font-size`, inserting DOM nodes | High |
| **Paint (repaint)** | Style → Paint → Composite | `background-image`, `color`, `box-shadow`, `border-radius` | Medium |
| **Composite only** | Style → Composite | `transform`, `opacity` (when promoted to their own layer) | Low |

web.dev calls the composite-only path "the cheapest and most desirable pathway through the pixel pipeline for high pressure points in a page's lifecycle, such as animations or scrolling." If you can animate it with `transform` or `opacity`, do.

> For a canonical list of which property triggers what, consult the per-property docs on MDN. Browser internals change — any printed "cheat sheet" ages fast.

## Main Thread vs Compositor Thread

The **main thread** runs JavaScript, style, layout, and paint. It is also where your app logic lives, so heavy work on it blocks everything.

The **compositor thread** assembles already-painted layers and pushes frames to the GPU. It runs independently of the main thread. That's why web.dev notes: "Chromium optimizes scrolling of the page so that it occurs solely on the compositor thread where possible, meaning that even if a page is not responding, you're still able to scroll the page and see parts of it that were previously drawn to the screen."

Consequence: animating `transform` / `opacity` on an element that has been promoted to its own compositor layer can keep running smoothly even if the main thread is busy. Animating `top` / `left` cannot — it forces layout on the main thread every frame.

## `will-change` and Layer Promotion

`will-change` is a hint to the browser: "this property is about to change, set up optimizations now." Typical values: `transform`, `opacity`, `scroll-position`, `contents`.

```css
.modal {
  will-change: transform, opacity;
}
```

MDN is blunt about misuse:

> `will-change` is intended to be used as a last resort, in order to try to deal with existing performance problems. It should not be used to anticipate performance problems.

Rules of thumb from the MDN page:

- Don't apply it to many elements — the optimizations (often a dedicated layer) cost memory.
- Prefer toggling `will-change` from JavaScript right before the change and removing it after, rather than hardcoding it in CSS.
- Give the browser a little time between setting it and animating — it's a hint, not a command.

Also note: setting `will-change` on properties like `opacity` can create a new stacking context, which can have visible side effects.

## `requestAnimationFrame` vs `requestIdleCallback`

Both schedule work, but for opposite situations.

| API | Fires when | Use for |
|-----|-----------|---------|
| **`requestAnimationFrame(cb)`** | Before the next repaint, synchronized with the display's refresh rate | Animations, anything that updates what the user sees this frame |
| **`requestIdleCallback(cb)`** | During an idle period on the main thread; may be deferred for seconds unless a `timeout` is set | Non-critical background work (analytics batching, prewarming caches) |

`requestAnimationFrame` is one-shot — call it again from inside the callback to keep animating. Its callback receives a `DOMHighResTimeStamp`; use it (or `performance.now()`) to compute progress, otherwise animations will run faster on high-refresh-rate displays (MDN).

`requestIdleCallback` has **limited browser availability** per MDN (not Baseline — Safari in particular). Provide a `timeout` option if the work must eventually run, or use a fallback.

```js
// Animation: runs every frame
function tick(ts) {
  element.style.transform = `translateX(${(ts / 10) % 200}px)`;
  requestAnimationFrame(tick);
}
requestAnimationFrame(tick);

// Background: runs when the browser is idle
requestIdleCallback(
  (deadline) => {
    while (deadline.timeRemaining() > 0 && queue.length) {
      processOne(queue.shift());
    }
  },
  { timeout: 2000 },
);
```

## Long Tasks and the 50ms Threshold

A **long task** is "any uninterrupted period where the main UI thread is busy for 50 milliseconds or longer" (MDN, `PerformanceLongTaskTiming`). Long tasks delay input handling, cause janky scrolling, and degrade Core Web Vitals like **INP** (Interaction to Next Paint), which web.dev grades as "good" at 200ms or less.

You can observe them directly:

```js
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn('Long task', entry.duration, 'ms', entry);
  }
});
observer.observe({ type: 'longtask', buffered: true });
```

> The Long Tasks API is still flagged experimental on MDN with limited support. Treat it as a diagnostic tool, not a production dependency.

## Layout Thrashing (and the Fix)

**Layout thrashing** is the classic main-thread killer: in a loop, you write a style, then read a layout property, which forces the browser to run layout *right now* to return an accurate value (a **forced synchronous layout** / forced reflow). Repeat this N times, get N layouts per frame.

The rule from web.dev is simple: **batch reads, then batch writes.**

```js
// BAD — reads offsetWidth every iteration after a style-affecting context,
// forcing a synchronous layout each time.
function resizeAllBad(box, paragraphs) {
  for (let i = 0; i < paragraphs.length; i++) {
    paragraphs[i].style.width = `${box.offsetWidth}px`;
  }
}

// GOOD — one read, then writes. No forced layouts inside the loop.
function resizeAllGood(box, paragraphs) {
  const width = box.offsetWidth; // read once
  for (let i = 0; i < paragraphs.length; i++) {
    paragraphs[i].style.width = `${width}px`; // writes only
  }
}
```

For multi-step UI updates, a common pattern is: queue reads, then queue writes inside `requestAnimationFrame` so all mutations hit the same frame.

## Reading the DevTools Performance Panel

In Chrome DevTools, the **Performance** panel records a timeline of the pipeline. For interview purposes you should be able to name what you're looking at:

- **Main** track — tasks on the main thread. Red corner tags mark long tasks (over 50ms).
- **Purple "Layout" / "Recalculate Style" / green "Paint" / teal "Composite Layers"** — the pipeline stages, so you can see which ones an interaction actually triggered.
- **Frames** track — per-frame timing. Frames dropping below your display's refresh rate show up as stretched bars.
- **Interactions** track — input events and the latency until the next paint (used to diagnose **INP**).

> Official docs live under Chrome DevTools documentation; feature names change occasionally, so don't memorize the UI — memorize the pipeline and you'll recognize it under any label.

## When to Reach for What

| Symptom | First thing to try |
|---------|--------------------|
| Animation stutters when main thread is busy | Animate `transform` / `opacity` instead of layout properties |
| A specific, about-to-animate element needs preparation | `will-change` toggled from JS right before animating, removed after |
| Heavy work blocking input | Break it up; move pieces to `requestIdleCallback` (with `timeout`) or a Web Worker |
| Scroll janky on mobile | Check for non-passive `touchstart` / `touchmove` listeners; ensure composited layers for animated elements |
| Layout trashed inside a loop | Batch reads, then writes |

---

[Next: DOM and Events →](02-dom-and-events.md) | [Back to index](README.md)
