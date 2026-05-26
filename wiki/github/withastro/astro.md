# withastro/astro

> The web framework for content-driven websites — ship less JavaScript by default.

[GitHub repo](https://github.com/withastro/astro) ·
[Official website](https://astro.build) ·
[License: MIT](https://github.com/withastro/astro/blob/main/LICENSE)

## Overview

Astro is a web framework whose central claim is **zero JavaScript by default**: pages render to static HTML at build time, and interactivity is opted into per-component via "islands"[^1]. This inverts the default of most modern frameworks, which ship a JavaScript runtime to every page and then optimize it away.

The second distinctive property is **framework-agnostic islands**: a single Astro project can mix React, Vue, Svelte, Solid, Preact, and Lit components on the same page, each rendered server-side and optionally hydrated client-side with their own runtime. This makes Astro a useful migration path for teams stuck between framework choices.

Astro 5 (2025-01) introduced **content layer** (a unified content API replacing the older content collections), **server islands** (deferred-render holes punched into static pages), and stable **sessions**[^2]. The framework has progressively become more capable of dynamic server rendering, but the design center remains content-heavy sites: docs, blogs, marketing, portfolios.

## Getting Started

```bash
npm create astro@latest my-site
cd my-site
npm install
npm run dev
```

A minimal Astro page with a React island:

```astro
---
import Counter from '../components/Counter.jsx'
const title = "Hello"
---

<html>
  <body>
    <h1>{title}</h1>
    <Counter client:visible />
  </body>
</html>
```

Add a framework integration:

```bash
npx astro add react
npx astro add tailwind
```

## Architecture / How It Works

Astro is built on Vite. At build time, `.astro` files are parsed into a JSX-like AST, components are executed server-side (in Node, Deno, or Bun), and the output is concatenated HTML. Framework components (`<ReactComponent />`) are rendered via each framework's SSR API (`react-dom/server`, `vue/server-renderer`, `svelte/server`) and inlined into the HTML.

**Client directives** control hydration:
- `client:load` — hydrate immediately on page load.
- `client:idle` — hydrate during the next idle callback.
- `client:visible` — hydrate when the component scrolls into view (IntersectionObserver).
- `client:media` — hydrate only when a media query matches.
- `client:only` — skip SSR, render only on the client.

Each island ships its own framework runtime. This is the largest tradeoff in Astro's design: a page with one React island and one Vue island ships both React and Vue. Sharing state across islands is intentionally awkward — `nanostores` is the recommended path[^3], but the framework discourages cross-island coupling.

**Server islands** (Astro 5) flip the model: a page is mostly static, with specific slots rendered on each request and streamed in after the static HTML. This is similar in spirit to React Server Components but framework-agnostic.

**Content collections** are typed Markdown/MDX/JSON directories with Zod schemas. The 5.0 content layer extended this to arbitrary loaders (CMS APIs, filesystem, anything).

## Production Notes

**Bundle behavior.** The "zero JS" promise holds for pages with no client islands. The moment you add `client:load` on a React component, you ship React + react-dom + the component code. Audit each island carefully.

**Multi-framework cost.** Mixing React and Vue on the same page is supported but produces ~150 KB of combined framework runtime. Use this for migrations, not as a default architecture.

**Static vs SSR.** Astro defaults to static output. Switching to SSR requires choosing an adapter (`@astrojs/node`, `@astrojs/vercel`, `@astrojs/cloudflare`, `@astrojs/netlify`). Adapter capabilities differ — Cloudflare's edge runtime is more restricted than Node.

**View Transitions.** Astro's `<ViewTransitions />` component wraps the browser View Transitions API and provides persistence across navigations. It works well for content sites but is not a substitute for a real SPA router.

**Markdown / MDX.** Remark and rehype plugin ecosystem applies. MDX support is via `@astrojs/mdx`. Shiki is the default syntax highlighter; large codeblocks can balloon build times.

**Image optimization.** `astro:assets` provides build-time image optimization via Sharp. SSR adapters need Sharp available at runtime, which fails on edge runtimes — use `@astrojs/cloudflare`'s image service or compile-time only.

## When to Use / When Not

**Use when:**
- Content-heavy: blogs, docs, marketing, portfolios.
- You want to migrate incrementally between frameworks (mix React + Vue + Svelte during a transition).
- You need maximum Lighthouse / Core Web Vitals scores by default.
- You're building a content-driven site with structured content collections.

**Avoid when:**
- The site is mostly an interactive app (admin dashboard, SaaS UI). Use Next.js, SvelteKit, or Nuxt instead.
- You need a thick client-side router with persistent state across navigations.
- Your team is unfamiliar with multiple frameworks and you'd be picking Astro just to "try them all" — the multi-framework story is a tool, not a goal.

## Alternatives

- [vercel/next.js](../vercel/next.js.md) — React-only, larger app-focused, better fit for interactive products.
- [sveltejs/svelte](../sveltejs/svelte.md) — SvelteKit covers similar ground for Svelte-only projects.
- [vuejs/core](../vuejs/core.md) — Nuxt is the equivalent Vue framework.
- [solidjs/solid](../solidjs/solid.md) — SolidStart for Solid-only projects.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2021-06 | Initial release by Fred K. Schott (Snowpack team). |
| 1.0 | 2022-08 | Stable, islands architecture[^1]. |
| 2.0 | 2023-01 | Content collections, hybrid rendering. |
| 3.0 | 2023-08 | View Transitions, image service. |
| 4.0 | 2023-12 | Dev toolbar, i18n routing. |
| 5.0 | 2025-01 | Content layer, server islands, stable sessions[^2]. |

## References

[^1]: Fred K. Schott, "Introducing Astro: Ship Less JavaScript" — 2021-06-08. https://astro.build/blog/introducing-astro/
[^2]: Astro Team, "Astro 5.0" — 2025-01-15. https://astro.build/blog/astro-5/
[^3]: Astro docs, "Sharing state between islands". https://docs.astro.build/en/recipes/sharing-state-islands/
