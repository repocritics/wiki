# TanStack/router

> A type-safe React router where the URL — routes, path params, and search params — is inferred end to end by the TypeScript compiler.

[GitHub repo](https://github.com/TanStack/router) ·
[Official website](https://tanstack.com/router) ·
[License: MIT](https://github.com/TanStack/router/blob/main/LICENSE)

## Overview

TanStack Router is a client-side router for React (with a Solid adapter and a
framework-agnostic core) whose defining bet is that the entire URL is a typed
API. Routes, path params (`$postId`), and — unusually — search params are all
statically inferred, so `navigate`, `Link`, `useParams`, and `useSearch` fail at
compile time when a route or param does not exist. It is authored by Tanner
Linsley, who also maintains TanStack Query, Table, and Form; Router is the
successor to his earlier React Location experiment and reached a stable 1.0 in
late 2023[^1].

The project ships two things from one repo. **TanStack Router** is the routing
library. **TanStack Start** is a full-stack meta-framework built on top of it —
SSR, streaming, and server functions — analogous to what Remix is to React
Router[^2]. This page is primarily about Router; Start inherits its routing model
and most of its tradeoffs.

The central tension is type safety versus tooling weight. Full inference of a
route tree is only possible because a code-generation step materializes a typed
`routeTree.gen.ts` from your route files, and because TypeScript is asked to
resolve very large inferred types. The payoff is that refactors and typos surface
in the editor; the cost is a build plugin in the loop and TypeScript memory/CPU
pressure on big route trees. Router also treats **search params as first-class
typed state** — validated, serialized, and subscribed to like any store — which
is its most distinctive design decision and the reason teams pick it over React
Router.

## Getting Started

```bash
npm install @tanstack/react-router
npm install -D @tanstack/router-plugin
```

Enable file-based routing through the Vite plugin (it generates
`routeTree.gen.ts` on save):

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import { tanstackRouter } from '@tanstack/router-plugin/vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [
    tanstackRouter({ target: 'react', autoCodeSplitting: true }),
    react(),
  ],
})
```

```tsx
// src/routes/posts.$postId.tsx
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  // search params are parsed + validated into typed state
  validateSearch: (search) => ({ page: Number(search.page ?? 1) }),
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
})

function PostComponent() {
  const post = Route.useLoaderData()          // typed from loader
  const { page } = Route.useSearch()          // typed from validateSearch
  return <h1>{post.title} — page {page}</h1>
}
```

Code-based routing (`createRoute` + `createRouter`) exists as an alternative for
teams that dislike the codegen step, at the cost of more manual wiring.

## Architecture / How It Works

**Route tree, generated.** File-based routing does not work by runtime glob
scanning. The `router-plugin` (Vite/Rspack/webpack/esbuild) watches `src/routes/`
and writes a single `routeTree.gen.ts` that imports every route module and stitches
them into a typed tree. `createRouter({ routeTree })` consumes it, and a global
`Register` interface declaration is how `Link`/`navigate` become aware of every
path. Without the generated file (or an equivalent hand-written tree), the type
safety does not exist — the codegen is load-bearing, not a convenience.

**Search params as state.** Each route can declare `validateSearch` (hand-rolled,
Zod, Valibot, or ArkType via adapters). The parsed result is typed, and updates
go through the same middleware/serialization path, so the URL query string behaves
like a validated, subscribable store rather than a string bag. This is the feature
with no clean equivalent in React Router.

**Loaders and caching.** Routes have `loader` functions with a built-in
SWR-style cache (`staleTime`, `gcTime`, preloading on intent/hover). The cache is
deliberately shallow — the docs recommend delegating real server-state to TanStack
Query and using loaders mainly to kick off / await queries, rather than treating
the router cache as a data layer.

**Nested layouts and boundaries** are expressed by file convention (`__root.tsx`,
pathless `_layout` routes, `route.lazy.tsx` for code splitting) with per-route
`pendingComponent` / `errorComponent` and Suspense-based transitions.

**TanStack Start** layers a server runtime on top: server functions
(`createServerFn`) compiled into RPC endpoints, full-document SSR + streaming, and
a Nitro/Vite-based build so it deploys to Node and various edge platforms.

## Production Notes

**TypeScript is the real cost center.** The end-to-end inference that makes Router
attractive also makes the compiler work hard. Large route trees (hundreds of
routes) can noticeably slow `tsc`, editor IntelliSense, and raise `tsserver` memory.
Mitigations that teams actually use: keep `validateSearch` return types explicit
rather than deeply inferred, split into smaller route groups, and stay current —
several releases have specifically targeted type-instantiation performance.

**The codegen step is a dependency, not optional.** CI must run the plugin (or a
committed `routeTree.gen.ts`) before typecheck, or types resolve to stale/`any`.
The generated file is a common merge-conflict source; most teams gitignore it and
regenerate, but that means a clean checkout has broken types until the dev server
or a generate command runs once.

**Migration churn.** The pre-1.0 betas moved fast and broke APIs; even post-1.0,
Router and especially Start have shipped renamed exports and plugin entry points
(e.g. the Vite plugin's export name changed over time). Pin versions and read
release notes before minor bumps.

**Start is younger than Router.** Router 1.x is mature and widely deployed; Start
reached its 1.0 line more recently and its server-function / deployment surface is
still stabilizing relative to Next.js's. Treat Start as production-capable but
earlier on the maturity curve than the router underneath it.

**SSR is opt-in and non-trivial.** Router alone is client-first; server rendering
requires Start (or manual SSR wiring). Do not assume SSR "for free" the way you
might with a framework-first tool.

## When to Use / When Not

**Use when:**
- Type-safe navigation and typed, validated **search params** are worth a build
  plugin in your pipeline.
- You have a genuinely SPA-shaped app with complex nested routing and URL-as-state.
- You already live in the TanStack ecosystem (Query especially) and want loaders
  that hand off to it.
- You want a client-first router now and the option to grow into full-stack via
  Start later.

**Avoid when:**
- You want a framework-first, SSR-by-default full-stack tool today — Next.js is
  more mature for that.
- Your route tree is huge and your team is sensitive to TypeScript build/IDE
  performance.
- You want zero code generation and minimal moving parts — a small hook router is
  simpler.
- You need the largest ecosystem of tutorials/hiring familiarity — React Router
  still dominates there.

## Alternatives

- remix-run/react-router — the incumbent; use it when ecosystem size and SSR
  maturity matter more than compile-time route/search-param type safety.
- vercel/next.js — use it when you want an RSC-first, framework-first full-stack
  app on a supported host, rather than Router + Start.
- molefrog/wouter — use it when you want a ~2KB hook-based router with no build
  step and no typed route tree.
- solidjs/solid-router — use it when you're on Solid and don't need TanStack's
  cross-framework model (TanStack Router does ship a Solid adapter too).
- remix-run/remix — folded into React Router 7; its loader/action model is the
  closest conceptual cousin to TanStack Start.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-01 | Repository origin; predates the current Router codebase[^3]. |
| beta | 2023 | Rapid pre-1.0 iteration; file-based routing + typed search params take shape. |
| 1.0 | 2023-12 | First stable release of TanStack Router[^1]. |
| Start (beta) | 2024 | Full-stack framework on top of Router enters public beta[^2]. |
| Start 1.0 | 2025 | Start reaches a 1.0 release line (see issues — exact date unverified). |

## References

[^1]: TanStack Router documentation and releases. https://tanstack.com/router
[^2]: TanStack Start documentation. https://tanstack.com/start
[^3]: Repository metadata, GitHub API `repos/TanStack/router` (created_at 2019-01-14). https://github.com/TanStack/router

## Tags

react, router, typescript, type-safe, routing, search-params, ssr, full-stack, spa, frontend, javascript
