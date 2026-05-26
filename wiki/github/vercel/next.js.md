# vercel/next.js

> The React framework for production — and the de facto reference implementation of React Server Components.

[GitHub repo](https://github.com/vercel/next.js) ·
[Official website](https://nextjs.org) ·
[License: MIT](https://github.com/vercel/next.js/blob/canary/LICENSE.md)

## Overview

Next.js is a React framework first released by Vercel (then Zeit) in 2016[^1]. For most of its life it was "React with file-system routing and SSR"; since version 13 (2022) it has been the lead implementation of React Server Components, server actions, and partial prerendering[^2]. As of 2026 it is the most opinionated, most-deployed React framework, and the one in which "React 19 features" are first usable in production.

Next.js is also tightly coupled to Vercel as a hosting platform. Many advertised features (Image Optimization, ISR with on-demand revalidation, Edge Functions, Middleware) are designed around Vercel's infrastructure; running the framework on other hosts (self-hosted Node, Cloudflare, AWS) requires explicit awareness of which features fall back to alternative implementations and which do not[^3]. This coupling is the most common production pain point in the Next.js ecosystem.

## Getting Started

```bash
npx create-next-app@latest my-app
# prompts: TypeScript? ESLint? Tailwind? src/ dir? App Router? Turbopack?
cd my-app
npm run dev
```

```tsx
// app/page.tsx — App Router, React Server Component by default
export default async function Page() {
  const data = await fetch("https://api.example.com/posts", {
    next: { revalidate: 60 },   // ISR — re-render at most once per 60s
  }).then(r => r.json());

  return (
    <ul>{data.map(p => <li key={p.id}>{p.title}</li>)}</ul>
  );
}
```

```tsx
// app/posts/new/actions.ts — Server Action
"use server";
import { redirect } from "next/navigation";
import { db } from "@/lib/db";

export async function createPost(formData: FormData) {
  await db.post.create({ data: { title: formData.get("title") as string } });
  redirect("/posts");
}
```

## Architecture / How It Works

Next.js has two coexisting routing systems:

1. **Pages Router** (`pages/`) — the original file-system router. Routes are React components, pages can opt into `getStaticProps` / `getServerSideProps` for data fetching. Stable, mature, will keep being supported per Vercel's communications, but is no longer where new features land[^2].
2. **App Router** (`app/`) — React Server Components as the default rendering unit. Files like `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `route.ts` define behavior by convention. Streaming is on by default.

Build tooling has shifted from webpack to **Turbopack**[^4] (Rust, by the same author as webpack). Turbopack is stable for `next dev` as of Next 15 and progressively rolling out for `next build`. The bundler change is mostly invisible but interacts with custom webpack config (which Turbopack does not consume).

Other notable internals:

- **`next/image`** — generates and caches resized images on the fly. The default loader writes to the host filesystem; on Vercel, it is replaced with the platform's image optimization service. Cache invalidation on self-hosted setups requires understanding of `next.config.js`'s `images` block.
- **Middleware** — runs in a constrained "edge runtime" (a subset of Web Standards APIs, no Node built-ins). On Vercel this maps to Edge Functions; on self-hosting it runs in a Node worker.
- **Server Actions** — RPC endpoints generated from `"use server"` functions. They are POST handlers under the hood with implicit CSRF protection via origin check.
- **Partial Prerendering (PPR)** — experimental in 14, GA path in 15: static shell + streamed dynamic holes within a single page.

The bundler, runtime, and route conventions are tightly co-evolved with React's own roadmap. This is the framework's biggest strength (newest React features land here first) and its biggest risk (breaking conventions ship before they are externally documented).

## Production Notes

**Self-hosting is supported but underdocumented.** A `next build && next start` will work for most use cases. Caveats:

- ISR with `revalidate` writes cache files to `.next/cache`. On multi-instance deploys this needs a shared cache (filesystem, Redis via `@neshca/cache-handler`, or similar)[^5].
- `next/image` optimization runs in-process and can become a CPU bottleneck. Production-grade self-hosting typically offloads to imgproxy / Cloudflare Images / S3 + Lambda.
- Middleware uses the edge runtime even on Node deployments, which means standard `fs` / `crypto` Node APIs are unavailable inside middleware.

**Build performance.** Large App Router apps can be slow to build; type-checking with TypeScript is often the dominant cost, not bundling. The standard mitigations are: incremental TypeScript builds (`tsc --build`), `SKIP_TYPECHECK=1` in CI with separate `tsc --noEmit` step, and Turbopack adoption.

**Cold starts.** On Vercel and similar serverless platforms, every route segment compiles into a separate function by default. RSC streaming masks but does not eliminate the cost of cold starts; expect 100–500 ms penalty for first-byte on under-trafficked routes.

**Common upgrade pains:**
- Next 12 → 13 introduced the App Router (opt-in); most teams ran both routers in parallel during migration.
- Next 13 → 14 stabilized server actions and changed the caching defaults.
- Next 14 → 15 changed the default for `fetch` from "cached" to "uncached"[^6] — a silent semantic break for code that relied on implicit caching.
- React 19 became required in Next 15; some third-party libraries lagged on peer-dep ranges for months.

**Auth.** There is no official auth solution. `next-auth` (now Auth.js)[^7] is the most common community option. Server actions + JWT in HttpOnly cookies is a working hand-rolled pattern. Clerk, Supabase Auth, Lucia, and WorkOS are alternative ecosystem options.

## When to Use / When Not

**Use when:**
- You want React Server Components without assembling them yourself.
- You're targeting Vercel and want the supported happy path.
- You need first-class i18n, image optimization, ISR, middleware, and a hosting story without piecing them together.
- You want a single framework that covers static, SSR, ISR, streaming, and edge in one config.

**Avoid when:**
- You're allergic to vendor coupling: many features have a "good on Vercel / awkward elsewhere" character.
- You're shipping a content-heavy mostly-static site: Astro is leaner and faster to build.
- You don't need React: SvelteKit and SolidStart cover similar ground with less runtime.
- You need a long-term-stable framework with rare breaking changes: Next.js iterates fast; major version cadence has been ~yearly with non-trivial migration steps.

## Alternatives

- [vitejs/vite](../vitejs/vite.md) + React Router — bring-your-own framework path; lighter, fewer opinions.
- [withastro/astro](../withastro/astro.md) — islands architecture, multiple UI frameworks, content-site-first.
- [sveltejs/svelte](../sveltejs/svelte.md) / SvelteKit — comparable framework story without React.
- [solidjs/solid](../solidjs/solid.md) + SolidStart — signals-based React-alternative with similar routing model.
- Remix (now part of React Router 7, not yet in this wiki) — Vercel's main framework-level competitor pre-2024.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-10-25 | Initial release. SSR + file-system routing[^1]. |
| 5.0 | 2018-02 | Dynamic imports, webpack 4. |
| 9.0 | 2019-07 | API routes, automatic static optimization. |
| 10.0 | 2020-10 | `next/image`, i18n routing. |
| 12.0 | 2021-10 | Middleware, Rust-based SWC compiler. |
| 13.0 | 2022-10 | App Router (beta), Turbopack alpha, server components[^2]. |
| 14.0 | 2023-10 | Server Actions stable, Partial Prerendering preview. |
| 15.0 | 2024-10 | Turbopack dev stable, React 19, `fetch` cache default change[^6]. |

## References

[^1]: Guillermo Rauch, "Next.js: Universal JavaScript Apps" — 2016-10-25. https://vercel.com/blog/next
[^2]: Next.js blog, "Next.js 13" — 2022-10-25. https://nextjs.org/blog/next-13
[^3]: Next.js docs, "Deploying — Self-hosting". https://nextjs.org/docs/app/building-your-application/deploying
[^4]: Turbopack documentation. https://turbo.build/pack
[^5]: `@neshca/cache-handler` — Next.js shared cache for self-hosted ISR. https://github.com/caching-tools/next-shared-cache
[^6]: Next.js blog, "Next.js 15" — 2024-10-21. https://nextjs.org/blog/next-15
[^7]: Auth.js (formerly NextAuth.js). https://authjs.dev/
