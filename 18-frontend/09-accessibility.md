# Web Accessibility (a11y)

Accessibility is a functional requirement, not a design polish. Around 15% of the world's population lives with some form of disability, and building an inaccessible product locks them out. In most jurisdictions it is also a legal requirement: the EU's **European Accessibility Act (EAA)** applied from June 2025, and in the US the ADA, Section 508, and EN 301 549 (by reference) all effectively point at WCAG.

## WCAG 2.2 — the standard

**Web Content Accessibility Guidelines 2.2** is the current W3C Recommendation, first published on **5 October 2023**. It's organized around four principles (**POUR**) and three conformance levels.

### POUR — four principles

| Principle | Meaning |
|-----------|---------|
| **Perceivable** | Information must be presentable in ways users can perceive (text alternatives for images, captions for audio, sufficient contrast) |
| **Operable** | UI and navigation must be operable (keyboard access, no seizure-inducing flashes, enough time to read) |
| **Understandable** | Information and UI operations must be comprehensible (readable text, predictable behavior, clear errors) |
| **Robust** | Content must work with current and future assistive technologies (valid HTML, correct ARIA, programmatic names/roles/states) |

### Conformance levels

| Level | Typical meaning |
|-------|-----------------|
| **A** | Minimum — failing these makes content unusable for some users |
| **AA** | **The target for legal compliance.** EAA, ADA settlements, Section 508, and EN 301 549 all point here |
| **AAA** | Aspirational — not recommended as a whole-site target by the W3C itself |

WCAG 2.2 added nine new success criteria over 2.1, mostly covering focus visibility (2.4.11–2.4.13), pointer interactions (2.5.7 Dragging Movements, 2.5.8 Target Size), and authentication/forms (3.2.6 Consistent Help, 3.3.7 Redundant Entry, 3.3.8–9 Accessible Authentication).

## Rule 1: semantic HTML first

The native element gives you keyboard handling, focus, ARIA role, and screen-reader text for free. ARIA can only describe semantics — it can't implement behavior.

```html
<!-- WRONG: ~20 lines of JS just to restore what <button> already does -->
<div role="button" tabindex="0" onclick="submit()"
     onkeydown="if(event.key==='Enter'||event.key===' ')submit()">
  Submit
</div>

<!-- RIGHT: keyboard, focus ring, Enter/Space, announced as "button" — all free -->
<button type="submit">Submit</button>
```

MDN's canonical warning:

> "If you can use a native HTML element or attribute with the semantics and behavior you require already built in, instead of re-purposing an element and adding an ARIA role, state or property to make it accessible, then do so."

And the common saying: **"No ARIA is better than bad ARIA"** (MDN). WebAIM's 2024 Million Report found that home pages with ARIA averaged 41% more detected errors than those without — because ARIA is often added to fix the wrong problem.

## ARIA essentials

### First rule of ARIA
Don't use it if a native element already does the job. Use it when no native equivalent exists (combobox, tree, tab panel, disclosure with unusual behavior) or to annotate state changes.

### Landmark roles (usually from semantic HTML)

| Element | Implicit role |
|---------|---------------|
| `<main>` | `main` |
| `<header>` (top-level) | `banner` |
| `<nav>` | `navigation` |
| `<aside>` | `complementary` |
| `<footer>` (top-level) | `contentinfo` |
| `<section>` with accessible name | `region` |

### Naming, description, state

| Attribute | Purpose |
|-----------|---------|
| `aria-label` | Direct, invisible label — use when there's no visible text (icon-only buttons) |
| `aria-labelledby` | Label by reference to visible element(s) via ID — preferred when a label is on screen |
| `aria-describedby` | Extra description (hints, error messages) that supplements the name |
| `aria-expanded` | `true` / `false` on disclosure triggers (accordions, menu buttons, combobox) |
| `aria-hidden="true"` | Hide purely decorative content from AT (never put it on a focusable element) |
| `aria-required`, `aria-invalid` | Form-field state (prefer native `required` where possible) |

### Live regions for dynamic updates

```html
<!-- Non-urgent: screen reader finishes current sentence first -->
<div aria-live="polite" id="status">Loading...</div>

<!-- Urgent: interrupts the current announcement. Use sparingly. -->
<div aria-live="assertive" id="error">Payment failed.</div>
```

Use `polite` for "saved", "loading complete", toast-style notifications. Reserve `assertive` for errors users must act on immediately.

## Keyboard navigation

Every interactive control must be reachable and operable with the keyboard alone — no mouse, no touch.

### Tab order

- Visit elements in **document order**. If your DOM order doesn't match your visual order, fix the DOM (or CSS), not with `tabindex`
- `tabindex="0"` — include an element in the natural tab order (for custom widgets)
- `tabindex="-1"` — focusable only programmatically (via `.focus()`); used for modal containers, skip-link targets, error summaries
- **Avoid positive `tabindex`.** MDN: "Avoid using `tabindex` values greater than `0` [...]. Doing so makes it difficult for people who rely on using keyboard for navigation or assistive technology to navigate and operate page content."

### Focus management on route changes (SPA)

In a traditional page navigation the browser moves focus and announces the new page. In an SPA, nothing happens unless you wire it up. The standard pattern:

```ts
// Angular: move focus + announce on navigated
import { inject } from '@angular/core';
import { Router, NavigationEnd } from '@angular/router';
import { filter } from 'rxjs';

const router = inject(Router);
const announcer = inject(LiveAnnouncer); // from @angular/cdk/a11y

router.events.pipe(filter(e => e instanceof NavigationEnd)).subscribe(() => {
  const h1 = document.querySelector<HTMLElement>('main h1');
  h1?.setAttribute('tabindex', '-1');
  h1?.focus();
  announcer.announce(`Navigated to ${document.title}`, 'polite');
});
```

### Visible focus indicators

WCAG 2.4.7 (AA) requires a visible focus indicator. **Do not** write `*:focus { outline: none }` without providing a replacement. WCAG 2.2's new 2.4.11 Focus Not Obscured (Minimum) also forbids overlays/sticky headers from hiding the focused element.

### Skip links

First focusable element on the page, visible on focus, jumps past navigation straight to content:

```html
<a href="#main" class="skip-link">Skip to main content</a>
<!-- ... nav ... -->
<main id="main" tabindex="-1">...</main>
```

## Screen readers

Test with at least one. Same markup behaves differently across combinations — a well-formed page on NVDA + Firefox can still trip up on VoiceOver + Safari.

| Screen reader | Platform | Cost |
|---------------|----------|------|
| **NVDA** | Windows | Free (open source) |
| **JAWS** | Windows | Paid (enterprise standard) |
| **VoiceOver** | macOS / iOS | Built in |
| **TalkBack** | Android | Built in |
| **Narrator** | Windows | Built in |

A senior-level a11y review always includes at least one real screen-reader run-through of critical flows (login, checkout, primary task).

## Color contrast

| Criterion | Ratio | Applies to |
|-----------|-------|------------|
| **1.4.3 Contrast (Minimum)** — AA | **4.5:1** body text, **3:1** for large text (18pt / 14pt bold) | All text |
| **1.4.6 Contrast (Enhanced)** — AAA | 7:1 body, 4.5:1 large | All text |
| **1.4.11 Non-text Contrast** — AA | **3:1** | UI components and meaningful graphics (icons, form borders, focus indicators) against adjacent colors |

Disabled controls are exempt from 1.4.11 — but if you rely on a grey button to signal "disabled", add an icon or aria-disabled so it isn't color-only (1.4.1 Use of Color).

## Accessible forms

```html
<!-- 1. Explicit label association -->
<label for="email">Email</label>
<input
  id="email"
  type="email"
  required
  aria-describedby="email-hint email-error"
  aria-invalid="true"
>
<p id="email-hint">We'll never share it.</p>
<p id="email-error" role="alert">Enter a valid email address.</p>

<!-- 2. Group related controls -->
<fieldset>
  <legend>Shipping method</legend>
  <label><input type="radio" name="ship" value="std"> Standard</label>
  <label><input type="radio" name="ship" value="exp"> Express</label>
</fieldset>
```

Checklist:

- Every input has a `<label for>` or `aria-labelledby` — placeholder text is **not** a label
- `required` on the native input (browser validates, AT announces it)
- Errors live in the DOM before submit if possible; associate with the field via `aria-describedby`
- Use `role="alert"` (or `aria-live="assertive"`) for errors that appear after submit
- Group radios/checkboxes with `<fieldset>` + `<legend>`
- Don't disable the submit button until the form is valid — let users try and read the errors

## Media

- **Captions** for all pre-recorded video with audio (WCAG 1.2.2 — AA)
- **Transcripts** for audio-only content (WCAG 1.2.1 — A)
- **Audio descriptions** for video where visual info is not in the dialogue (WCAG 1.2.5 — AA)
- Never autoplay audio; if you must, provide a visible pause/mute

## Automated tooling — necessary but not sufficient

| Tool | What it is |
|------|------------|
| **axe-core** (Deque) | Open-source rules engine; powers axe DevTools, Lighthouse a11y, Pa11y, Playwright a11y tests |
| **Lighthouse (a11y)** | In Chrome DevTools; runs a subset of axe-core rules |
| **Pa11y** | CLI wrapper around axe/HTML_CodeSniffer for CI pipelines |
| **WAVE** (WebAIM) | Browser extension; visualizes errors inline |

**How much do they catch?** axe-core's README states it finds "on average 57% of WCAG issues automatically" — but treat that as an upper bound. Public audits (WebAIM Million) report only the issues their automated scanner detects and explicitly warn: "All automated tools, including WAVE, have limitations—not all conformance failures can be automatically detected. Absence of detected errors does not indicate that a page is accessible or conformant." The commonly cited range is **~30–57% automated coverage, varying by study and rule set**.

> Practical conclusion: automated tools catch the easy stuff (missing alt, missing label, low contrast, empty button). They cannot judge whether alt text is *meaningful*, whether focus order makes sense, whether a custom widget is usable with a screen reader, or whether your flow works with voice control. Every product flow still needs manual keyboard + screen-reader review.

## Senior interview checklist

- [ ] WCAG 2.2 AA is the target; POUR is the mental model
- [ ] Native HTML before ARIA; `<button>` not `<div role="button">`
- [ ] All interactive controls reachable by keyboard; visible focus indicator
- [ ] Avoid positive `tabindex`; move focus on SPA route changes
- [ ] Every form input has a label; errors are associated and announced
- [ ] Color contrast 4.5:1 text, 3:1 UI and large text
- [ ] Automated a11y testing (axe, Lighthouse) in CI + manual screen-reader pass for core flows
- [ ] Announce async updates with `aria-live` (polite by default)

---

[← Previous: SPA vs SSR vs SSG](08-spa-ssr-ssg.md) | [Next: Promises vs Observables →](10-promises-vs-observables.md) | [Back to index](README.md)
