# Web Performance

Web performance is what decides whether a user stays or bounces. For a senior frontend interview, you are expected to know **Core Web Vitals**, how to **measure** them (lab vs field), and the concrete **optimization levers** you pull to move each metric.

## Core Web Vitals

Google's **Core Web Vitals** are the three user-centric metrics that Chrome uses as the canonical health signal for a page. Each is assessed at the **75th percentile** of page loads, separately for mobile and desktop.

| Metric | Measures | Good | Needs improvement | Poor |
|--------|----------|------|-------------------|------|
| **LCP** (Largest Contentful Paint) | Loading — when the largest content element in the viewport is rendered | ≤ 2.5 s | 2.5 s – 4.0 s | > 4.0 s |
| **INP** (Interaction to Next Paint) | Responsiveness — time from user interaction until the next frame is painted | ≤ 200 ms | 200 – 500 ms | > 500 ms |
| **CLS** (Cumulative Layout Shift) | Visual stability — largest burst of unexpected layout shift scores | ≤ 0.1 | 0.1 – 0.25 | > 0.25 |

> **INP replaced FID as a stable Core Web Vital in 2024.** Per web.dev, "support for FID ended on September 9, 2024. You should now focus on INP." Unlike FID, which only measured the first interaction's input delay, INP observes all clicks, taps and key presses for the full page lifecycle and reports the longest (minus outliers).

### What counts for LCP

LCP considers `<img>`, `<image>` inside `<svg>`, `<video>` (poster or first frame), elements with a CSS `background-image` loaded via `url()`, and block-level elements containing text. Invisible elements, full-viewport backgrounds and low-detail placeholders are filtered out.

## Supporting metrics

These are not Core Web Vitals but are essential for diagnosing them.

| Metric | What it is | Good threshold |
|--------|-----------|---------------|
| **TTFB** (Time to First Byte) | Time from navigation start until the first response byte arrives. Diagnoses server/CDN/redirect cost | ≤ 0.8 s |
| **FCP** (First Contentful Paint) | When *any* content (text, image, SVG, non-white canvas) first renders | ≤ 1.8 s |
| **TBT** (Total Blocking Time) | Lab-only — sum of blocking time between FCP and TTI. Proxy for INP in lab | N/A (lab proxy) |
| **Speed Index** | Lab-only — how quickly the visible page content is populated | N/A (lab proxy) |

> TTFB is a "rough guide" per web.dev, not a Core Web Vital. SSR-heavy sites legitimately trade TTFB for LCP.

## Lab data vs field data

| | Lab | Field (RUM) |
|---|-----|-------------|
| Collected from | Synthetic run in a controlled environment | Real users in the wild |
| Tools | Lighthouse, WebPageTest, DevTools | **CrUX** (Chrome UX Report), `web-vitals` JS, PageSpeed Insights field section |
| Strengths | Reproducible, good for debugging and CI gates | Reflects actual user experience (network, device, geography) |
| Weaknesses | Not representative of real users — cannot measure **INP** reliably (no real interactions) | Noisy; needs traffic volume; harder to attribute regressions |

> **INP cannot be reliably measured in the lab** because labs don't produce realistic user interactions. Use TBT as a lab proxy, and measure INP in the field.

## Lighthouse

Lighthouse is Chrome's open-source audit tool. It runs a set of checks across **four categories**: **Performance, Accessibility, SEO, Best Practices**. (The older "PWA" category has been retired from the core report.)

### Lighthouse CI

**Lighthouse CI** ([GoogleChrome/lighthouse-ci](https://github.com/GoogleChrome/lighthouse-ci)) runs Lighthouse on every PR, stores historical results and fails the build on regressions. The typical pipeline:

```yaml
# .github/workflows/lhci.yml
- name: Run Lighthouse CI
  run: |
    npm install -g @lhci/cli
    lhci autorun --upload.target=temporary-public-storage
```

Paired with **assertions** in `lighthouserc.json`, you get PR-level gates like "LCP must be ≤ 2.5 s on /product/\*".

## Field measurement — the `web-vitals` library

[`web-vitals`](https://github.com/GoogleChrome/web-vitals) is the official Google library for measuring Core Web Vitals in production. It is tiny (~2 KB brotli) and matches how Chrome computes the metrics.

```typescript
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics({ name, value, id, rating }) {
  navigator.sendBeacon('/analytics', JSON.stringify({ name, value, id, rating }));
}

onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);
```

The `attribution` build surfaces **why** a metric is bad (the specific element for LCP, the specific event target for INP), which is what you want for root-cause analysis.

> CrUX is the public dataset that powers PageSpeed Insights' "Field Data" section and the Core Web Vitals report in Search Console. It is aggregated, so you still want your own RUM pipeline (via `web-vitals`) for per-session diagnosis.

## Optimization techniques

### Images — usually the LCP element

- **Responsive images** via `srcset` and `sizes` so mobile doesn't download a 4K hero.
- **Modern formats**: WebP and AVIF typically save 25–50% over JPEG/PNG at equivalent quality (vendor-reported; verify on your own assets).
- **`loading="lazy"`** on below-the-fold images to skip off-screen downloads.
- **`fetchpriority="high"`** on the LCP image to tell the browser "this one first." Per MDN, `fetchpriority` provides a hint — `high`, `low`, or `auto` — for the relative priority of the fetch.
- **Preload** the LCP image with `<link rel="preload" as="image" href="hero.avif" fetchpriority="high">` when the browser can't discover it early (e.g., CSS background).

```html
<img
  src="hero-800.avif"
  srcset="hero-400.avif 400w, hero-800.avif 800w, hero-1600.avif 1600w"
  sizes="(max-width: 600px) 400px, 800px"
  fetchpriority="high"
  alt="..." />
```

> **Never** put `loading="lazy"` on the LCP image — you will delay the paint that *defines* LCP.

### Fonts

- `font-display: swap` — per MDN, this gives the font an "extremely small block period and an infinite swap period," rendering the fallback immediately and swapping in the custom font when it loads. Prevents invisible text (FOIT) but can cause a visible swap (FOUT).
- **Subset** the font (remove unused glyphs) — large wins on CJK/icon fonts.
- **Preload** critical fonts: `<link rel="preload" as="font" type="font/woff2" href="/font.woff2" crossorigin>`. The `crossorigin` attribute is required for fonts even on same-origin.

### Critical CSS

Inline the CSS needed to render the above-the-fold content in a `<style>` tag in `<head>`, and load the rest async. Eliminates the render-blocking CSS round trip.

### HTTP caching and protocols

- **Cache-Control** with long `max-age` + hashed filenames (`app.8f3a1.js`) — "immutable" caching for static assets.
- **HTTP/2** — multiplexing of many small requests over one connection. Kills the old "concatenate everything" advice.
- **HTTP/3 / QUIC** — UDP-based, better on lossy networks, 0-RTT for reconnects.

### Resource hints

| Hint | Does what | Cost | Use for |
|------|----------|------|---------|
| `preload` | Fetches a resource for the **current** navigation at high priority | Full download | LCP image, critical font, late-discovered JS/CSS |
| `prefetch` | Fetches a resource for a **future** navigation at idle priority | Full download, low priority | Next-page JS bundle |
| `preconnect` | Opens DNS + TCP + TLS to an origin ahead of time | Connection only | Critical third-party origin (CDN, API) |
| `dns-prefetch` | Resolves DNS only | Cheapest | Many third-party origins you might use |

> Overusing `preconnect` hurts more than it helps — MDN: "If a page needs to make connections to many third-party domains, preconnecting them all can be counterproductive." Reserve it for critical origins.

### JavaScript

- **Code-splitting**: per-route chunks via dynamic `import()`. The initial bundle only carries what the first view needs.
- **Tree-shaking**: bundlers eliminate unused exports. Requires ES modules (`import`/`export`) and side-effect-free code — CommonJS defeats it.

## Performance budgets

A **performance budget** is a hard ceiling you set for a metric; CI fails when you exceed it. Lighthouse supports this via **LightWallet** with a `budget.json` file — three kinds of budgets:

```json
[
  {
    "path": "/*",
    "timings": [
      { "metric": "largest-contentful-paint", "budget": 2500 },
      { "metric": "interactive", "budget": 3500 }
    ],
    "resourceSizes": [
      { "resourceType": "script", "budget": 170 },
      { "resourceType": "total", "budget": 500 }
    ],
    "resourceCounts": [
      { "resourceType": "third-party", "budget": 10 }
    ]
  }
]
```

Run with `lighthouse <url> --budget-path=./budget.json`. Results appear in the Budgets section of the Performance category.

## Interview checklist

- Know the **three CWV thresholds** cold: LCP 2.5 s, INP 200 ms, CLS 0.1.
- Know that **INP replaced FID on September 9, 2024**.
- Know the **75th-percentile rule** — CWVs are evaluated at p75 per device class.
- Know the **lab vs field** distinction and why INP needs the field.
- Be able to describe the **LCP playbook**: preload the LCP image, `fetchpriority="high"`, no `lazy` on it, ship AVIF/WebP with `srcset`, inline critical CSS, trim blocking JS.

---

[← Previous: Web Storage](05-web-storage.md) | [Next: PWA and Service Workers →](07-pwa-service-workers.md) | [Back to index](README.md)
