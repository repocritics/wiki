# QwikDev/qwik

> A resumable JavaScript framework that serializes application state into HTML to avoid hydration, shipping almost no JavaScript on initial load.

[GitHub repo](https://github.com/QwikDev/qwik) ·
[Official website](https://qwik.dev) ·
[License: MIT](https://github.com/QwikDev/qwik/blob/main/LICENSE)

## Overview

Qwik is a component framework created by Miško Hevery — the original author of AngularJS — and incubated at Builder.io[^1]. Its one distinguishing idea is *resumability*: instead of re-executing component code on the client to attach event listeners and rebuild state (the "hydration" step every other SSR framework performs), Qwik serializes the framework's state, listeners, and component boundaries directly into the server-rendered HTML. The browser can then "resume" the application from where the server left off, downloading component code only when a specific interaction demands it.

The practical target is time-to-interactive on large pages. A Qwik page can become interactive with a few kilobytes of framework code regardless of application size, because nothing is eagerly hydrated. This is a genuinely different tradeoff from the React/Next.js model (reduce client JS via server components, but still hydrate islands) and from Astro (ship zero JS by default, opt into islands). Qwik keeps a single component model for the whole page and defers *everything*.

The cost of this design is a mental model and a set of constraints that do not exist in other frameworks. The `$` sigil, the serialization rules, and the many-small-chunks network profile are the defining tensions of working in Qwik, and they surface in production more than in demos. The ecosystem is also considerably smaller than React's — with roughly 22k GitHub stars as of 2026 it is an established but niche framework, not a default choice.[^2]

## Getting Started

```sh
npm create qwik@latest
# scaffolds a Qwik + Qwik City (router) app on Vite
```

```tsx
// src/components/counter.tsx
import { component$, useSignal } from "@builder.io/qwik";

export default component$(() => {
  const count = useSignal(0);
  // onClick$ is a lazy boundary: this handler is NOT in the initial bundle.
  // It is fetched on first click and the app "resumes" without re-running
  // the component that rendered it.
  return (
    <button onClick$={() => count.value++}>
      Count: {count.value}
    </button>
  );
});
```

The `$` suffix (`component$`, `onClick$`, and the bare `$()` wrapper) marks a boundary the Qwik optimizer is allowed to extract into a separately loadable chunk. This is not decorative syntax — it is the instruction to the build tool.

## Architecture / How It Works

Qwik has three layers that are usually discussed together but are separable:

1. **Core runtime** (`@builder.io/qwik`) — the component model, signals-based reactivity (`useSignal`, `useStore`), and the resumability machinery. State is stored in serializable proxies; the runtime tracks which listeners and subscriptions exist so they can be written into HTML.
2. **Optimizer** — a Rust transform (built on SWC) that runs during the Vite build. Every `$`-suffixed expression is hoisted into its own exported symbol and replaced with a QRL (a serializable, lazy reference of the form `chunk.js#symbol`). This is what turns one source file into dozens of independently loadable fragments.[^3]
3. **Qwik City / Qwik Router** — the meta-framework: file-system routing, `routeLoader$`/`routeAction$` for server data, endpoints, and deployment adapters. In the Qwik 2 line this is being renamed to Qwik Router and the package scope moves to `@qwik.dev/*`.[^4]

Resumability works because the QRL is a string. When the server renders, event handlers are emitted as attributes like `on:click="./chunk.js#handler_a"`. A tiny global listener installed once on the document intercepts the first event of each type, reads the QRL, dynamically imports the chunk, and invokes the handler. No component tree is walked on load; there is no hydration pass. Reactivity is fine-grained: updating a signal re-runs only the specific closures subscribed to it, not a component render.

The consequence is that code-splitting is not a manual optimization you apply later — it is the default behavior of the compiler, at the granularity of individual event handlers and render fragments.

## Production Notes

**Serialization is the sharp edge.** Any state captured across a `$` boundary must be serializable by Qwik. Closures that capture class instances, non-plain objects, functions that are not themselves QRLs, or circular structures Qwik cannot handle will fail — sometimes at runtime rather than build time. Teams coming from React repeatedly hit this when they try to close over a library object or a callback inside an event handler.

**Many small chunks change the network profile.** The same design that keeps initial JS tiny produces a long tail of small module requests as the user interacts. Qwik mitigates this with a service-worker-based prefetcher and speculative module preloading driven by usage data, but the profile still assumes HTTP/2+ and benefits heavily from a CDN. On high-latency connections the per-interaction fetch can be perceptible if prefetching has not primed the relevant chunk.

**SSR is not optional for the value proposition.** Resumability only pays off when the page is server-rendered (or statically generated) so the serialized state exists in the HTML. Running Qwik as a pure client-side SPA discards the entire reason to use it.

**React interop reintroduces hydration.** `@builder.io/qwik-react` lets you mount React components inside Qwik, which is useful for reaching the React library ecosystem, but those islands hydrate the normal (eager) way and carry React's runtime cost. It is an escape hatch, not a free bridge.

**Debugging crosses the optimizer.** Because handlers are hoisted into generated chunks, stack traces and breakpoints do not map cleanly to source without source maps, and the indirection of QRLs makes "where does this run" harder to answer than in a framework that keeps handlers inline.

**Upgrade pain: the Qwik 2 migration.** The move from the `@builder.io/qwik` scope to `@qwik.dev/core`, the Qwik City → Qwik Router rename, and associated API adjustments make the 1.x → 2.x transition a real migration rather than a version bump; pin versions and read the migration notes before upgrading.[^4]

## When to Use / When Not

**Use when:**
- Time-to-interactive on content-heavy, interactive pages is a measured priority and hydration cost is your bottleneck.
- You control SSR/SSG and can deploy behind a CDN with HTTP/2+.
- You want automatic, compiler-driven code splitting rather than hand-tuned lazy imports.
- The team can absorb a new mental model (`$`, resumability, serialization rules).

**Avoid when:**
- You depend on a large React/Vue component library surface — interop exists but with caveats and re-hydration cost.
- Your app is a mostly-static content site — Astro ships less framework code for that shape.
- You need a large hiring pool, deep third-party ecosystem, or abundant Stack Overflow answers; Qwik is comparatively niche.
- You cannot server-render, or your state is inherently non-serializable.

## Alternatives

- marko-js/marko — eBay's framework that pioneered resumable server rendering; closest philosophical peer, more mature in that specific lineage.
- solidjs/solid — signals-based fine-grained reactivity with excellent runtime performance, but uses conventional hydration.
- withastro/astro — use when the site is content-first and you want zero JS by default with opt-in islands.
- sveltejs/svelte — compiler approach with a small runtime; use when you want minimal client JS without the resumability constraints.
- vercel/next.js — use when you want to reduce client JS via React Server Components and value ecosystem depth over TTI extremes.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2022 | Public previews and alphas while incubated at Builder.io.[^1] |
| 1.0 | 2023-05 | First stable release; resumability and Qwik City declared production-ready.[^5] |
| 1.x | 2023–2025 | Ongoing 1.x line; optimizer, prefetching, and adapter improvements. |
| 2.0 (beta) | 2024–2026 | Qwik 2 line: `@qwik.dev/*` package scope, Qwik City renamed Qwik Router, insights-driven preloading.[^4] |

## References

[^1]: Builder.io / Miško Hevery on Qwik's origins and resumability. https://www.builder.io/blog/hydration-is-pure-overhead
[^2]: GitHub repository metadata (stars, forks, license, last push) fetched via the GitHub API, 2026-07. https://github.com/QwikDev/qwik
[^3]: Qwik docs, "Progressive / precision lazy-loading" and the optimizer. https://qwik.dev/docs/concepts/progressive/
[^4]: Qwik documentation and package scopes (`@qwik.dev/core`, Qwik Router). https://qwik.dev/
[^5]: Qwik docs, "Resumable vs Replayable." https://qwik.dev/docs/concepts/resumable/

## Tags

typescript, javascript, web-framework, frontend, resumability, ssr, jsx, signals, code-splitting, vite, meta-framework, hydration
