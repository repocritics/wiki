# sveltejs/kit

> The official application framework for Svelte — file-system routing, SSR, and adapter-based deployment on top of Vite.

[GitHub repo](https://github.com/sveltejs/kit) ·
[Official website](https://svelte.dev/docs/kit) ·
[License: MIT](https://github.com/sveltejs/kit/blob/main/LICENSE)

## Overview

SvelteKit is the first-party application framework for Svelte, the same way Next.js is for React or Nuxt for Vue. Svelte is a compiler that turns `.svelte` components into imperative DOM code; SvelteKit is the layer above it that supplies routing, server-side rendering, data loading, form handling, and a build pipeline. It reached 1.0 in December 2022 after a long public beta, replacing the older Sapper framework[^1].

Its defining architectural decision is the **adapter model**. A SvelteKit build produces a platform-agnostic intermediate artifact, and a per-platform *adapter* transforms that into something deployable — a Node server (`adapter-node`), a static site (`adapter-static`), or serverless/edge bundles for Vercel, Netlify, or Cloudflare[^2]. This is deliberately different from Next.js, whose best features assume Vercel's runtime. SvelteKit's bet is portability; the tradeoff is that any given platform's happy path depends on the maturity of its community-or-first-party adapter, and behavior can differ subtly between them.

The second defining trait is that SvelteKit is built on **Vite**[^3]. It is not a bundler itself — dev server, HMR, and production builds (via Rollup) all come from Vite. In practice this means most build issues in a SvelteKit project are actually Vite issues, a distinction the maintainers call out explicitly in the bug tracker.

## Getting Started

```bash
npx sv create my-app      # scaffolds a SvelteKit project
cd my-app
npm install
npm run dev
```

```svelte
<!-- src/routes/+page.svelte — a route is a directory with a +page.svelte -->
<script>
  let { data } = $props();   // Svelte 5 runes; data comes from load()
</script>

<ul>
  {#each data.posts as post}
    <li>{post.title}</li>
  {/each}
</ul>
```

```js
// src/routes/+page.server.js — runs only on the server
export async function load({ fetch }) {
  const posts = await fetch('/api/posts').then((r) => r.json());
  return { posts };   // must be serializable — SvelteKit uses devalue
}
```

## Architecture / How It Works

Routing is **file-system based** under `src/routes/`, but unlike most frameworks the routing files are distinguished by prefix, not by being "the component":

- `+page.svelte` — the page component (rendered on server then hydrated on client).
- `+page.js` — a *universal* `load` (runs on server during SSR, then on client during navigation).
- `+page.server.js` — a *server-only* `load` plus form `actions`; never shipped to the browser.
- `+layout.svelte` / `+layout.js` / `+layout.server.js` — nested layouts and their data.
- `+server.js` — a standalone HTTP endpoint (GET/POST/…), i.e. an API route.
- `+error.svelte` — error boundary for a route subtree.

The **universal vs. server `load`** distinction is the single most load-bearing concept and the most common source of confusion: universal `load` can return non-serializable values (like class instances) but runs in the browser too, so it cannot touch secrets or a database; server `load` can, but its return value must be serializable because it crosses the network. Data flows down through the layout tree and merges.

**Form actions** are the server-side mutation primitive. A `<form method="POST">` posts to a named action in `+page.server.js`; the `use:enhance` action upgrades it to a fetch-based submission without a full page reload, and the whole thing degrades to a plain HTML form if JavaScript is unavailable. This progressive-enhancement stance is a core design value, not an afterthought.

**Rendering is configurable per route** via exported constants: `export const ssr = false` (SPA mode), `export const csr = false` (no client JS at all), `export const prerender = true` (render to static HTML at build time). A single app can mix prerendered marketing pages, SSR'd app pages, and client-only routes.

**Environment variables** are exposed through four virtual modules — `$env/static/private`, `$env/dynamic/private`, `$env/static/public`, `$env/dynamic/public` — where `static` is inlined at build time and `public` requires a `PUBLIC_` prefix. The compiler will error if a private env var leaks into client-reachable code, which is a genuine safety improvement over ad-hoc `process.env` access.

## Production Notes

**Adapters are not interchangeable.** `adapter-auto` detects Vercel/Netlify/Cloudflare/etc. in CI and picks for you, but for any production deployment you control you should pin an explicit adapter — `adapter-auto` has no configuration surface and silently does nothing on unrecognized platforms. Self-hosting means `adapter-node`, which emits a standalone Node server you run yourself.

**Serverless/edge constraints leak upward.** On `adapter-vercel` or `adapter-cloudflare`, code in server `load`/endpoints runs in a constrained runtime — Cloudflare Workers have no Node built-ins, edge functions have execution-time and API limits. Code that works under `adapter-node` can fail on an edge adapter, and the failure surfaces at deploy or request time, not build time.

**Prerendering has sharp edges.** `prerender = true` requires that every prerendered page be reachable by crawling links from an entry point, or listed in `config.kit.prerender.entries`. Dynamic routes with no incoming link are silently missed. Pages that read request headers or cookies cannot be prerendered and will error.

**Data must be serializable.** Server `load` return values and action results are serialized with `devalue` (which handles `Date`, `Map`, `Set`, `BigInt`, cyclic references — more than JSON) but not class instances, functions, or streams. Returning an ORM entity object frequently fails here.

**The Svelte 5 migration was significant.** Svelte 5 (October 2024) introduced *runes* (`$state`, `$props`, `$derived`, `$effect`), a signals-based reactivity model that replaced the compiler-magic `let`/`$:` reactivity of Svelte 3/4[^4]. SvelteKit works across the boundary, but application code, component libraries, and documentation examples split between the two dialects for a long time. Check whether a third-party Svelte library supports runes before adopting it.

**SvelteKit 2.0 (December 2023) had breaking changes** — it required Vite 5 and Svelte 4+, changed how `redirect`/`error` helpers are thrown vs. returned, and adjusted cookie defaults[^5]. The upgrade is scriptable but not zero-effort.

## When to Use / When Not

**Use when:**
- You want Svelte with a real routing/SSR/data-loading story rather than assembling one.
- Deployment portability matters — you may move between Node, static hosting, and serverless.
- You value progressive enhancement and small client bundles (Svelte ships less runtime than React).
- You want prerender/SSR/SPA modes mixable per route in one project.

**Avoid when:**
- Your team is committed to the React ecosystem — component libraries and hiring pool are far larger there.
- You need a specific platform's most advanced primitives (e.g. deep Vercel ISR/image tooling) that Next.js supports first-class.
- You are shipping a purely static content site — Astro builds faster and ships less JS for that shape.
- You need long-term API stability with rare breaking changes; SvelteKit and Svelte have both had meaningful migrations recently.

## Alternatives

- vercel/next.js — the React equivalent; larger ecosystem, more vendor coupling. Use it when the team is React-first.
- nuxt/nuxt — the Vue equivalent with a comparable module/adapter model. Use it when your components are Vue.
- withastro/astro — content-first islands architecture that can embed Svelte components. Use it for mostly-static, content-heavy sites.
- solidjs/solid + SolidStart — signals-based fine-grained reactivity with a similar full-stack story. Use it when you want Solid's performance model.
- remix-run/react-router — file-based full-stack React with a comparable loader/action design. Use it when you want SvelteKit's ergonomics but in React.

## History

| Version | Date | Notes |
|---------|------|-------|
| beta | 2020-10 | Public repo opens; early betas built on Snowpack. |
| beta | 2021-05 | Switched build tooling from Snowpack to Vite[^3]. |
| 1.0 | 2022-12-14 | First stable release; supersedes Sapper[^1]. |
| 2.0 | 2023-12-14 | Vite 5 requirement, shallow routing, `redirect`/`error` changes[^5]. |
| — | 2024-10 | Svelte 5 (runes) ships; SvelteKit supports both dialects[^4]. |

## References

[^1]: Svelte blog, "Announcing SvelteKit 1.0" — 2022-12-14. https://svelte.dev/blog/announcing-sveltekit-1.0
[^2]: SvelteKit docs, "Adapters". https://svelte.dev/docs/kit/adapters
[^3]: Vite — the build tool SvelteKit is built on. https://vitejs.dev/
[^4]: Svelte docs, "Runes" (Svelte 5). https://svelte.dev/docs/svelte/what-are-runes
[^5]: Svelte blog, "SvelteKit 2.0". https://svelte.dev/blog/sveltekit-2

## Tags

javascript, svelte, sveltekit, web-framework, ssr, full-stack, vite, file-based-routing, frontend, meta-framework
