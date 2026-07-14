# bigskysoftware/htmx

> Access AJAX, WebSockets, and SSE directly from HTML attributes — server-rendered HTML fragments instead of a client-side JSON app.

[GitHub repo](https://github.com/bigskysoftware/htmx) ·
[Official website](https://htmx.org) ·
[License: 0BSD](https://github.com/bigskysoftware/htmx/blob/master/LICENSE)

## Overview

htmx is a ~14 KB (min+gzip), dependency-free JavaScript library that lets you
trigger HTTP requests from any element and any event, then swap the returned
HTML into the page — all declared via `hx-*` attributes[^1]. It is the successor
to Carson Gross's earlier jQuery-based intercooler.js, rewritten with no
dependencies, and is maintained by Big Sky Software[^2].

The premise is a deliberate rejection of the SPA default: rather than shipping
JSON to a client framework that renders it, the server renders HTML fragments
and htmx splices them into the DOM. The intellectual framing is HATEOAS and
Roy Fielding's REST — "completing HTML as a hypertext" by removing the arbitrary
limits that only `<a>`/`<form>` issue requests, only click/submit trigger them,
and only whole-page replacement is possible[^1]. This is elaborated at length in
the maintainers' book *Hypermedia Systems*[^3].

The defining tradeoff: htmx moves complexity back to the server. You trade the
client-side state machine (and its tooling, hydration, and bundle) for a
requirement that your backend produce partial HTML on demand and that request
flow, caching, and history be reasoned about at the HTTP layer. For CRUD-shaped,
server-authoritative apps this is a large simplification; for rich offline or
highly interactive client state it is a poor fit, and htmx does not pretend
otherwise.

## Getting Started

```html
<!-- CDN: pin an exact version and use SRI in production -->
<script src="https://cdn.jsdelivr.net/npm/htmx.org@2.0.4/dist/htmx.min.js"></script>

<!-- POST on click; replace the button itself with the response HTML -->
<button hx-post="/clicked" hx-swap="outerHTML">Click Me</button>
```

```bash
npm install htmx.org   # note: the package is htmx.org, NOT htmx (that one is broken)
```

The server responds with an HTML fragment, not JSON:

```html
<!-- response to POST /clicked -->
<button hx-post="/clicked" hx-swap="outerHTML">Clicked!</button>
```

Common attributes: `hx-get`/`hx-post`/etc (verb + URL), `hx-trigger` (which
event fires it), `hx-target` (where the response goes, defaults to the element),
and `hx-swap` (how — `innerHTML` by default, or `outerHTML`, `beforeend`,
`delete`, `none`, plus modifiers like `swap:200ms` and `scroll:top`).

## Architecture / How It Works

htmx is a single event-driven script that scans the DOM for `hx-*` attributes and
wires up listeners. On a trigger it issues an `XMLHttpRequest`, adds request
headers (`HX-Request`, `HX-Target`, `HX-Trigger`, `HX-Current-URL`), and on
response swaps the returned markup into the target, re-processing the new nodes so
their attributes are live too.

Coordination happens through **HTTP headers in both directions**. The response
can steer htmx without touching the body: `HX-Redirect`, `HX-Location`,
`HX-Push-Url`, `HX-Retarget` (change the target), `HX-Reswap` (change the swap
style), and `HX-Trigger` (fire client-side events by name). This header protocol
is the real API surface — the attributes are only the client half.

Beyond simple swaps: `hx-swap-oob` ("out of band") lets one response update
multiple disjoint parts of the page. `hx-boost` progressively enhances plain
`<a>`/`<form>` into AJAX navigations. `hx-push-url` integrates with the History
API, and htmx keeps a snapshot cache in the browser's history state so Back
restores prior fragments. Events (`htmx:beforeRequest`, `htmx:afterSwap`,
`htmx:responseError`, and many more) are the extension and debugging seam.

htmx 2.0 (2024) narrowed the core: WebSockets and Server-Sent Events, once
built in, are now separate extensions (`ws`, `sse`), Internet Explorer support
was dropped, and `htmx.config.selfRequestsOnly` defaults to `true`[^4]. The
companion languages `_hyperscript` and Alpine.js are frequently paired with htmx
for the client-side interactions it intentionally omits.

## Production Notes

- **Your server must emit HTML fragments.** This is an architectural commitment,
  not a config flag. Templating, partial rendering, and "is this a full page or a
  fragment?" branching (often on the `HX-Request` header) become backend
  concerns. Teams underestimate how much this reshapes the server.
- **XSS is your responsibility.** htmx inserts response HTML into the DOM. Any
  untrusted data rendered into a fragment is an injection vector; server-side
  escaping/templating discipline is mandatory. `hx-disable` marks subtrees that
  must never be processed. There is no client framework auto-escaping to lean on.
- **`selfRequestsOnly` and CSP.** Since 2.0 htmx blocks cross-origin requests by
  default; calling other origins requires opting out. A strict CSP is compatible
  but interacts with inline event handling (`hx-on:*`) and any `eval`-based
  config — review `htmx.config` accordingly.
- **History cache footguns.** The Back-button snapshot lives in the browser
  history and can capture sensitive or stale DOM. Use `hx-history="false"` on
  pages that must not be cached, and be aware fragment responses and full pages
  can diverge on restore.
- **Focus, scroll, and swap scope.** Large `innerHTML`/`outerHTML` swaps discard
  and rebuild nodes, losing focus, scroll position, and any un-persisted client
  state inside the swapped region. Prefer targeted, small swaps or OOB updates;
  `hx-preserve` retains specific elements across swaps.
- **Caching.** GET fragments are cacheable like any URL; a cached fragment served
  where a full page is expected (or vice versa) is a classic bug — `Vary` on
  `HX-Request` if the same URL returns both.
- **Loading/UX state is manual.** htmx gives you `hx-indicator` and request
  classes, but debouncing, optimistic UI, and concurrent-request handling
  (`hx-sync`) must be composed explicitly — there is no reactive scheduler.

## When to Use / When Not

**Use when:**
- The app is server-authoritative CRUD/forms/dashboards where the server already
  owns rendering.
- You want to delete a client build/state layer and keep a small, stable JS dep.
- You value progressive enhancement and want plain HTML to work without JS.

**Avoid when:**
- You need rich offline behavior, heavy client-side state, or complex local
  interactivity (drawing, editors, real-time collaboration DOM).
- Your backend cannot easily produce HTML fragments (JSON-only API, or a static
  host with no server rendering).
- Your team is committed to a component model (design system in React/Vue) where
  server HTML would fragment ownership of the UI.

## Alternatives

- hotwired/turbo — use instead when you're on Rails/Hotwire and want frames and
  streams as a convention rather than per-attribute wiring.
- alpinejs/alpine — not a replacement; use alongside htmx for purely client-side
  state (dropdowns, tabs, toggles) that needs no server round-trip.
- unpoly/unpoly — use when you want a heavier, more prescriptive server-driven
  framework with built-in modals, layers, and optimistic UI.
- hotwired/stimulus — use when you only need to attach behavior to existing HTML
  and are not doing server-fragment swapping at all.
- starfederation/datastar — use when you want an SSE-first, signals-based take on
  hypermedia in a comparably small footprint.

## History

| Version | Date | Notes |
|---------|------|-------|
| intercooler.js | 2013– | jQuery-based predecessor by the same author[^2]. |
| 0.0.x | 2020 | Dependency-free rewrite; first public htmx releases[^2]. |
| 1.0 | 2020 | First stable line; `hx-*` attribute API established[^1]. |
| 1.9.x | 2023 | Mature 1.x; extensions ecosystem, view-transition support. |
| 2.0.0 | 2024-06 | IE support dropped; WS/SSE moved to extensions; `selfRequestsOnly` default true[^4]. |

## References

[^1]: htmx documentation — introduction and reference. https://htmx.org/docs/
[^2]: htmx repository (bigskysoftware/htmx), README and history; successor to intercooler.js. https://github.com/bigskysoftware/htmx
[^3]: C. Gross, A. Stepinski, D. Akşimşek, *Hypermedia Systems*. https://hypermedia.systems/
[^4]: htmx 2.0 release notes and migration guide. https://htmx.org/migration-guide-htmx-1/

## Tags

javascript, html, hypermedia, hateoas, ajax, frontend, server-rendered, progressive-enhancement, rest, no-build
