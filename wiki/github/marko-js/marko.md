# marko-js/marko

> An HTML-based UI language and compiler from eBay, built around streaming SSR and shipping as little client JavaScript as possible.

[GitHub repo](https://github.com/marko-js/marko) ·
[Official website](https://markojs.com) ·
[License: MIT](https://github.com/marko-js/marko/blob/main/LICENSE)

## Overview

Marko is a declarative, HTML-superset language that compiles `.marko` files to JavaScript for both server and client rendering. It originated inside eBay, was open-sourced in 2014, and still powers eBay's production storefront[^1]. Almost any valid HTML is valid Marko; the language then layers on components, control-flow tags (`<if>`, `<for>`), and a reactivity system, so authors write markup-first rather than JSX-first.

Its defining design goal is minimizing client-side JavaScript. Marko pioneered out-of-order streaming server rendering — the server flushes HTML as async data resolves rather than waiting for the full page — and automatic partial hydration, where the compiler statically determines which components are actually interactive and ships runtime + state only for those[^2]. The newer authoring model (the "Tags API", seen in the README's `<let/count=0>` counter) pushes this further with fine-grained reactivity and a resumability story that aims to eliminate the hydration pass entirely.

The tension around Marko is adoption, not capability. Technically it has shipped ideas — streaming SSR, partial hydration, resumability — years before mainstream React frameworks reached parity, but its mindshare is small, its ecosystem is thin, and it is effectively a single-company framework: eBay is both primary sponsor and primary user. Choosing Marko means learning a bespoke language and betting on a project whose center of gravity is one corporate backer[^1].

## Getting Started

```bash
npm init marko
```

A single-file component (`click-count.marko`) using the modern Tags API:

```marko
<let/count=0>
<button onClick() { count++ }>
  Clicked ${count} times
</button>
```

Markup, control flow, and interactivity live together; `<let>` declares reactive state, `${}` interpolates, and the `onClick() {}` shorthand attaches a handler. The compiler decides that only this component's interactivity needs to reach the browser.

## Architecture / How It Works

Marko is a **compiler first, runtime second**. The toolchain parses `.marko` templates into an AST and emits optimized JS render functions; there is no in-browser template interpreter. Two things fall out of the compile step:

1. **Streaming SSR.** The server render is a generator that flushes HTML incrementally. Async work (data fetches) does not block the whole document — the shell streams immediately and out-of-order fragments are patched in as promises resolve[^2]. This predates React Suspense streaming by years.
2. **Automatic partial hydration.** The compiler performs static analysis to classify components as stateless (server-only, zero client JS) or interactive. Only interactive subtrees receive runtime code and serialized state, so a mostly-static page can ship almost no JavaScript — an islands-style outcome derived automatically instead of hand-annotated.

Marko has carried **two authoring models**. The older **class/component API** (Marko 5 and earlier) used explicit component objects, lifecycle methods, and a virtual-DOM-based client runtime. The newer **Tags API** (the Marko 6 line) replaces that with lightweight tag-scoped variables (`<let>`, `<const>`, `<return>`) and a fine-grained reactivity graph, targeting **resumability** — serializing enough state into the HTML that the client can resume interaction without re-running the render to hydrate[^3]. This is the same problem space as Qwik, reached from an HTML-language direction.

Build integration is via **`@marko/vite`** (the Vite plugin) and the file-based meta-framework **`@marko/run`**, which supplies routing and a dev/build pipeline on top of Vite. Marko is not just a template library — the intended path is the full Marko + Vite + `@marko/run` stack.

## Production Notes

- **Ecosystem depth is the real constraint.** There is no large third-party component/library market. UI kits, auth integrations, and data libraries that "just work" in React often have to be wrapped or reimplemented. Budget for building things you would otherwise install.
- **The Tags API is a genuine rewrite, not a syntax refresh.** Moving from the class/component API to Tags API changes the reactivity mental model and the client runtime, not just the file syntax. Treat a 5→6 migration as a rewrite of authoring patterns, and pin your major version deliberately; do not assume old class-API examples map one-to-one onto Tags API code[^3].
- **Streaming interacts with your infrastructure.** Out-of-order streaming only pays off if the whole response path — reverse proxies, CDNs, serverless response buffering — actually streams. Platforms that buffer the full response before sending erase the first-byte advantage. Verify streaming end-to-end, not just in `next`-style local dev.
- **Partial hydration hides a footgun.** Because the compiler decides what is interactive, adding an event handler or reactive value to a previously static component silently pulls new runtime into the client bundle. Interactivity has a cost that is easy to add without noticing.
- **Hiring and knowledge base.** Fewer engineers know Marko than React/Vue, and there is proportionally less Stack Overflow / blog coverage. The official docs and Discord are the primary support surface; expect to read source for edge cases.
- **Maintenance is healthy but concentrated.** The repo is actively pushed and issues are triaged tightly (open-issue count stays low), but the contributor base is eBay-centric. Assess bus-factor risk accordingly.

## When to Use / When Not

**Use when:**
- You are building content-heavy or commerce pages where shipping minimal client JS materially improves load and conversion.
- Streaming SSR and automatic partial hydration are core requirements, not nice-to-haves.
- You want an HTML-first authoring model rather than JSX.
- You control the full response path and can guarantee real streaming to the client.

**Avoid when:**
- You depend on a broad third-party component/library ecosystem (React/Vue win decisively here).
- Your team needs easy hiring and abundant learning material.
- You want framework stability with no bespoke-language investment — Marko's two-API history means migration risk.
- Your app is a client-heavy SPA where server streaming and partial hydration buy you little.

## Alternatives

- sveltejs/svelte — compiler-based, "less runtime" philosophy with a far larger community; use instead when you want the compiled-framework approach with mainstream ecosystem support.
- BuilderIO/qwik — resumability-first framework; use when resumability/zero-hydration is the primary goal but you prefer a JSX-like, React-adjacent DX.
- solidjs/solid — fine-grained reactivity in JSX; use when you want Marko-like reactivity without adopting a bespoke HTML language.
- withastro/astro — islands architecture with explicit partial hydration; use for content sites where you want to mix UI frameworks and hand-pick interactive islands.
- vuejs/core — HTML-ish single-file templates with a deep ecosystem; use when you want markup-first templating plus mainstream tooling and hiring.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-sourced | 2014 | Released by eBay from its internal UI framework[^1]. |
| Marko 4 | 2017 | Major rewrite: new compiler and VDOM-based client runtime. |
| Marko 5 | 2021 | Modernized build/runtime; class/component API line. |
| Marko 6 (Tags API) | 2023–2026 | New compiler, fine-grained reactivity, resumability; the current authoring model[^3]. |

## References

[^1]: Marko is developed and maintained primarily by eBay and used in eBay's production web frontend. https://tech.ebayinc.com/ and https://markojs.com
[^2]: Marko documentation — rendering, streaming, and automatic partial hydration. https://markojs.com/docs/
[^3]: Marko Tags API and the Marko 6 direction (reactivity, resumability). https://markojs.com/docs/reference/reactivity

## Tags

javascript, ui-framework, html, compiler, server-side-rendering, streaming, partial-hydration, resumability, reactivity, frontend, ebay, isomorphic
