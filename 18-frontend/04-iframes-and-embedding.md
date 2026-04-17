# iframes and Cross-Origin Embedding

The `<iframe>` is the oldest browser primitive for embedding one document inside another. In modern frontends it shows up in three scenarios: **third-party widgets** (maps, videos, social), **isolation** (sandboxing untrusted HTML), and **cross-origin payment/auth flows** (Stripe Elements, 3-D Secure).

Get the security model wrong and you ship clickjacking, postMessage leaks, or XSS. The controls are: `sandbox`, `postMessage` origin checks, `X-Frame-Options` / CSP `frame-ancestors`, and COOP/COEP.

## When to use an iframe

| Scenario | Why iframes |
|----------|-------------|
| **Third-party widgets** (YouTube, Google Maps, ads) | Provider controls the JS; you don't audit it |
| **Payment forms** (Stripe Elements, 3DS) | PCI scope reduction — card input lives in the provider's frame |
| **Isolating untrusted content** (user-provided HTML, email preview) | Sandbox separates the DOM, storage, and scripting context |
| **Legacy widgets** (dashboards, embedded reporting) | No modular rewrite path |

**Not a good fit:** SPA composition (use Module Federation or Web Components), styling integration with parent (iframes are hard to size and theme).

## The `sandbox` attribute

Adding `sandbox` with no value applies **all restrictions**: no scripts, no forms, no popups, the document gets a unique opaque origin (fails same-origin policy against everyone, including its own server). You then re-enable specific capabilities token by token.

```html
<iframe src="https://untrusted.example.com/widget"
        sandbox="allow-scripts allow-forms"></iframe>
```

### Sandbox tokens

From MDN's `<iframe>` reference:

| Token | Effect |
|-------|--------|
| `allow-same-origin` | Without it, the document is treated as having a unique opaque origin and fails same-origin checks. |
| `allow-scripts` | Allows the page to run scripts (but not create pop-up windows). |
| `allow-forms` | Allows form submission. |
| `allow-popups` | Allows popups (`window.open`, `target="_blank"`). |
| `allow-popups-to-escape-sandbox` | Popups open without the parent's sandbox flags. |
| `allow-modals` | Allows `alert()`, `confirm()`, `print()`, `prompt()`. |
| `allow-pointer-lock` | Enables the Pointer Lock API. |
| `allow-presentation` | Allows starting a presentation session. |
| `allow-top-navigation` | Allows navigating the top-level browsing context. |
| `allow-top-navigation-by-user-activation` | Same, but only via a user gesture. |
| `allow-top-navigation-to-custom-protocols` | Allows navigating to non-HTTP protocols. |
| `allow-downloads` | Allows downloads via `<a download>` or navigation-to-download. |
| `allow-orientation-lock` | Allows screen orientation lock. |
| `allow-storage-access-by-user-activation` | Allows use of the Storage Access API to request unpartitioned cookies. |

> **The dangerous combination:** `sandbox="allow-scripts allow-same-origin"` with a frame from your own origin. The frame can script, access `document.cookie`, and remove the sandbox attribute from itself by navigating — the sandbox becomes ornamental. Use only for trusted content from your origin, or drop `allow-same-origin`.

### Minimal-permission recipe for untrusted HTML

```html
<iframe sandbox="allow-scripts"
        srcdoc="<p>User HTML here</p>"></iframe>
```

No `allow-same-origin` → the frame has an opaque origin and can't read the parent's cookies, storage, or DOM. No `allow-forms` → no silent data exfiltration via form POST. No `allow-top-navigation` → can't navigate the parent away.

## Same-origin policy and iframes

An iframe whose origin differs from the parent follows the **same-origin policy**:

- Parent **cannot** read `iframe.contentDocument`, `iframe.contentWindow.document`, or the iframe's `localStorage`.
- The iframe **cannot** read the parent's DOM or storage.
- They **can** communicate only via `postMessage`.

With `sandbox` (no `allow-same-origin`), the iframe's origin is **opaque** — distinct from every other origin, including the URL it was loaded from. That means the iframe can't read its own server's cookies even via `fetch`.

## `postMessage` — cross-origin communication

```javascript
// Parent -> iframe
iframe.contentWindow.postMessage(
  { type: 'init', token: 'xyz' },
  'https://widget.example.com'   // targetOrigin — NEVER '*' with secrets
);

// Iframe -> parent
window.parent.postMessage({ type: 'ready' }, 'https://app.example.com');

// Receiving (either side)
window.addEventListener('message', (event) => {
  // 1. Verify origin
  if (event.origin !== 'https://widget.example.com') return;
  // 2. Verify source (optional but safer)
  if (event.source !== expectedWindow) return;
  // 3. Validate schema
  if (typeof event.data?.type !== 'string') return;

  handleMessage(event.data);
});
```

### The `'*'` anti-pattern

MDN is explicit: **"Always provide a specific `targetOrigin`, not `*`, if you know where the other window's document should be located. Failing to provide a specific target could disclose data to a malicious site."** A malicious page can navigate the target window to its own origin without your knowledge; `'*'` then leaks to them.

### Mandatory receiver-side checks

MDN again: **"If you do not expect to receive messages from other sites, do not add any event listeners for `message` events."** And: **"always verify the sender's identity using the `origin` and possibly `source` properties"**, and **"always verify the syntax of the received message"**.

> `event.origin` is the sender's origin at the time `postMessage` was called — format is `scheme://host[:port]`, e.g. `https://example.org`. Treat it as untrusted input until you've string-compared it against a known allowlist.

## Framing defenses: `X-Frame-Options` vs CSP `frame-ancestors`

Both prevent **your** page from being framed by others (clickjacking defense). They are response headers on the embedded page.

### `X-Frame-Options`

```http
X-Frame-Options: DENY
X-Frame-Options: SAMEORIGIN
```

From MDN:

- `DENY`: "The document cannot be loaded in any frame, regardless of origin."
- `SAMEORIGIN`: "The document can only be embedded if all ancestor frames have the same origin as the page itself."
- `ALLOW-FROM origin`: **"This is an obsolete directive. Modern browsers that encounter response headers with this directive will ignore the header completely."**

MDN recommends CSP `frame-ancestors` over `X-Frame-Options` for more comprehensive control. `X-Frame-Options` is still widely deployed and supported in older browsers — ship both if you're defensive-in-depth.

### CSP `frame-ancestors`

```http
Content-Security-Policy: frame-ancestors 'none';
Content-Security-Policy: frame-ancestors 'self';
Content-Security-Policy: frame-ancestors 'self' https://trusted.partner.com;
```

MDN: "specifies valid parents that may embed a page using `<frame>`, `<iframe>`, `<object>`, or `<embed>`." Setting it to `'none'` is similar to `X-Frame-Options: deny`.

> **`frame-ancestors` is about who can embed *me*. `frame-src` is about what *I* can embed.** Different directive, different direction.

### Recommended default

```http
Content-Security-Policy: frame-ancestors 'none'
X-Frame-Options: DENY
```

Only relax if the page is intentionally embeddable.

## COOP and COEP

These two headers govern **cross-origin isolation** of your top-level document. They are required if you want to use `SharedArrayBuffer` or high-resolution `performance.now()`.

### `Cross-Origin-Opener-Policy` (COOP)

MDN: "allows a website to control whether a new top-level document, opened using `Window.open()` or by navigating to a new page, is opened in the same browsing context group (BCG) or in a new browsing context group."

| Value | Behavior |
|-------|----------|
| `unsafe-none` (default) | Shares BCG with any document. No isolation. |
| `same-origin` | Only same-origin `same-origin` documents share a BCG. Enables cross-origin isolation. |
| `same-origin-allow-popups` | Like `same-origin` but allows popups with `unsafe-none` to be opened in the same BCG. |
| `noopener-allow-popups` | Always opens into a new BCG unless the opener also has `noopener-allow-popups`. |

### `Cross-Origin-Embedder-Policy` (COEP)

MDN: "configures the current document's policy for loading and embedding cross-origin resources that are requested in `no-cors` mode."

| Value | Behavior |
|-------|----------|
| `unsafe-none` (default) | Allows cross-origin `no-cors` resources without permission. |
| `require-corp` | Cross-origin resources must explicitly opt in via `Cross-Origin-Resource-Policy`, or be requested in `cors` mode. |
| `credentialless` | Allows cross-origin `no-cors` resources, but requests are sent **without credentials** (no cookies, and cookies in the response are ignored). |

### Unlocking `SharedArrayBuffer`

```http
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

Both headers are required. Check at runtime with `crossOriginIsolated`.

```javascript
if (crossOriginIsolated) {
  const buffer = new SharedArrayBuffer(16);
}
```

> COEP `require-corp` will break embedding any third-party asset that doesn't set `Cross-Origin-Resource-Policy`. Plan the migration carefully, or use `credentialless`.

## `<iframe credentialless>`

MDN: "Set to `true` to make the `<iframe>` credentialless, meaning that its content will be loaded in a new, ephemeral context. It doesn't have access to the network, cookies, and storage data associated with its origin. It uses a new context local to the top-level document lifetime. In return, the Cross-Origin-Embedder-Policy (COEP) embedding rules can be lifted, so documents with COEP set can embed third-party documents that do not."

```html
<iframe credentialless src="https://widget.example.com"></iframe>
```

Use when you need cross-origin isolation (COEP `require-corp`) but still want to embed third parties that don't set CORP headers.

## Clickjacking: why framing defenses exist

A malicious page frames your bank's "Transfer $1000" page inside a transparent iframe over a decoy button. The user thinks they click "Claim prize"; they actually click inside your iframe. Framing defenses (`X-Frame-Options`, CSP `frame-ancestors`) block this at the browser level by refusing to render the page in a disallowed frame.

> Assume every authenticated page is a clickjacking target. Default to `frame-ancestors 'none'` and opt specific pages in.

## `referrerpolicy`

Controls what `Referer` header the frame sends.

```html
<iframe src="..." referrerpolicy="strict-origin-when-cross-origin"></iframe>
```

Default is `strict-origin-when-cross-origin`. For third-party widgets that leak path data, tighten to `no-referrer` or `origin`.

## `loading="lazy"`

```html
<iframe src="..." loading="lazy"></iframe>
```

MDN: "Defer loading of the iframe until it reaches a calculated distance from the visual viewport." Default is `eager`. Use `lazy` for below-the-fold embeds (YouTube, maps) — each lazy iframe saves a round-trip and the third-party's JS execution until it actually matters.

## Trust boundaries when embedding user content

| Boundary | Control |
|----------|---------|
| **User HTML in your domain** | `<iframe sandbox srcdoc="...">` — opaque origin, no scripts unless explicit |
| **Third-party widget** | Dedicated subdomain, strict CSP on parent, postMessage API with origin check |
| **Payment provider** | Provider-hosted iframe (Stripe Elements), parent never sees card data |
| **Embedding untrusted user on your real origin** | Don't. Serve user content from a **separate origin** (`usercontent.example.com`) so same-origin policy protects your main site |

> Shared-cookie-jar is the silent killer. A user-content iframe on your primary domain shares cookies with the parent — one XSS owns the session. Always use a dedicated origin for user-generated content.

---

[← Previous: CDN](03-cdn.md) | [Next: Web Storage →](05-web-storage.md) | [Back to index](README.md)
