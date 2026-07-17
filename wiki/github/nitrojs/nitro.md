# nitrojs/nitro

> Universal JavaScript server toolkit — write h3 handlers once, deploy the same
> app to Node, Cloudflare, Vercel, Netlify, Deno, Bun, or Lambda via presets.

[GitHub repo](https://github.com/nitrojs/nitro) ·
[Official website](https://nitro.build) ·
[License: MIT](https://github.com/nitrojs/nitro/blob/main/LICENSE)

## Overview

Nitro is the server engine extracted from the Nuxt 3 effort (repo created
January 2022, published as `nitropack`)[^1]. Its pitch is deployment
portability: you write file-based server routes against the h3 HTTP framework,
and a build-time **preset** compiles the app into a self-contained output for
a specific host — Node server, Cloudflare Worker, Vercel Build Output, Netlify
Function, Deno Deploy, Bun, AWS Lambda, and a few dozen more. Nitro 2.0 was
the first standalone major release in January 2023[^2]; the project later left
the `unjs` organization for its own `nitrojs` org during v3 development (the
old `unjs/nitro` URL redirects).

Nitro sits on the unjs library stack: h3 for request handling, unstorage for
key-value storage, unenv for runtime polyfills, and Rollup for server
bundling. Most users encounter it indirectly — it is Nuxt's server layer, and
it powered Vinxi, the meta-bundler behind early TanStack Start and SolidStart.
Standalone adoption is real but smaller; at ~11k stars it is actively
maintained (pushed within days as of mid-2026), with a long tail of ~570 open
issues[^1]. The defining tension: "deploy anywhere" is a build-time
abstraction over runtimes that genuinely differ. Presets normalize routing,
env access, and storage, but they cannot make workerd behave like Node — the
abstraction leaks exactly where production problems live.

## Getting Started

Nitro v2 (`nitropack` on npm) is the current stable; v3 is in beta[^3].

```bash
npx giget@latest nitro nitro-app --install
cd nitro-app
npm run dev
```

```ts
// routes/hello.ts — file-based routing, h3 handler, auto-imported helpers
export default defineEventHandler((event) => {
  const name = getQuery(event).name ?? "world";
  return { message: `hello ${name}` };
});
```

```bash
npm run build            # → .output/ (self-contained, deps bundled)
node .output/server/index.mjs
NITRO_PRESET=cloudflare_module npm run build   # retarget without code changes
```

## Architecture / How It Works

- **Handlers** — every route is an h3 `EventHandler`. Files under `routes/`
  and `api/` map to URL paths (`routes/users/[id].get.ts` → `GET /users/:id`);
  helpers like `defineEventHandler` and `readBody` auto-import via unimport.
- **Presets** — a preset is a Rollup config plus entrypoint glue plus output
  layout for one platform: `node-server` emits a listener script,
  `cloudflare_module` a Worker fetch handler, `vercel` the Build Output API
  directory. Platform selection is one env var.
- **Bundling** — server code is bundled with Rollup; unenv aliases Node
  built-ins to shims on non-Node targets, and `@vercel/nft` traces
  externalized `node_modules` for Node targets.
- **Storage & cache** — `useStorage()` exposes unstorage mounts (filesystem in
  dev; Redis, Cloudflare KV, S3 drivers in production);
  `defineCachedEventHandler` / `cachedFunction` build stale-while-revalidate
  caching on top of it[^4].
- **Route rules & prerendering** — declarative per-route config (`cache`,
  `prerender`, `redirect`, `headers`, `isr`) translated into platform features
  where they exist; a build-time crawler renders selected routes to static
  HTML in the same output.

Nitro v3 repositions the project as a Vite-native server layer ("extends your
Vite app with a production-ready server"), building on Vite's environment API
and the web-standards rewrite of h3[^3]. It has shipped as date-tagged betas
(e.g. `v3.0.260610-beta`) through 2026 while v2 continues on `nitropack`.

## Production Notes

**The preset abstraction leaks.** Code that passes `nitro dev` (Node) can fail
on workerd or Lambda: Node built-ins need `nodejs_compat` on Cloudflare, unenv
shims can silently no-op instead of throwing, and worker bundle-size limits
bite when Rollup inlines a heavy dependency. Test on the target platform in
CI, not just locally.

**Bundling server code is not free.** Packages with dynamic `require`, native
addons, or `__dirname`-relative file reads break under Rollup. The escape
hatches (`externals.inline`/`external`) work but turn into per-dependency
whack-a-mole; debugging means reading the generated `.output/` bundle.

**Cache defaults don't survive horizontal scaling.** The cache layer writes to
memory/filesystem by default; multi-instance or serverless deploys need an
explicit shared unstorage driver (e.g. Redis) or each instance keeps its own
cache. SWR background revalidation also depends on the platform keeping the
runtime alive after the response — behavior differs across providers.

**Route rules are provider-dependent.** `isr` maps to real infrastructure only
on providers that have it (Vercel, Netlify); elsewhere it degrades. Read the
per-preset docs rather than assuming rules are portable[^5].

**v2 → v3 is a real migration.** v3 changes the framing (Vite plugin), the h3
major underneath (v1 → v2, web-standards based), and the npm package name
(`nitropack` → `nitro`). The beta window has been long; teams starting today
on v2 should budget for API churn. Experimental v2 surfaces (WebSockets,
scheduled tasks) are flagged as such and have shifted between minors.

**Nuxt coupling cuts both ways.** Nuxt usage means Nitro is well-exercised,
but standalone questions get less community coverage, and some behavior is
easiest to find documented from the Nuxt side.

## When to Use / When Not

**Use when:**
- You want one server codebase deployable to multiple hosts, or expect to switch providers and want that to be a config change.
- You are on Nuxt (you already use it) or building a framework-level tool that needs a deployment-target abstraction.
- You want batteries — caching, storage, prerendering, route rules — without assembling five libraries.

**Avoid when:**
- You deploy to exactly one platform: its native tooling (Wrangler, plain Fastify on a VM) is simpler and leaks less.
- Your dependencies resist bundling (native addons, dynamic requires) — the Rollup step becomes your maintenance burden.
- You need a stable multi-year API surface right now: the v2 → v3 transition is still in flight.

## Alternatives

- h3js/h3 — use the handler/routing layer directly when you don't need Nitro's build, presets, or caching machinery.
- honojs/hono — use instead for a runtime-portable, router-first framework with no bundling pipeline and first-class Cloudflare support.
- fastify/fastify — use instead for Node-only long-lived servers where raw throughput and a mature plugin ecosystem matter more than portability.
- expressjs/express — use instead when ecosystem familiarity and middleware breadth outweigh performance and multi-runtime concerns.
- nksaraf/vinxi — Nitro + Vite meta-bundler behind earlier TanStack Start/SolidStart; largely superseded by Vite's environment API and Nitro v3.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x–1.x | 2022 | Developed inside the Nuxt 3 effort as `nitropack`; repo created 2022-01[^1]. |
| 2.0 | 2023-01-24 | First standalone major release[^2]. |
| 2.8–2.12 | 2023–2025 | Incremental v2 line: experimental WebSockets, tasks, preset expansion. |
| 2.13.4 | 2026-04-29 | Latest stable v2 (`nitropack` on npm)[^3]. |
| 3.0 beta | 2025–2026 | Date-tagged betas (`v3.0.260610-beta`); Vite-native rework; `nitrojs` org[^3]. |

## References

[^1]: GitHub API, nitrojs/nitro — 11,030 stars, 864 forks, 568 open issues, created 2022-01-26, pushed 2026-07-17. Retrieved 2026-07-17. https://github.com/nitrojs/nitro
[^2]: Nitro v2.0.0 release — 2023-01-24. https://github.com/nitrojs/nitro/releases/tag/v2.0.0
[^3]: npm: `nitropack` latest = 2.13.4; `nitro` = 3.0.260610-beta. README v3-branch notice. https://www.npmjs.com/package/nitropack
[^4]: Nitro docs, "Cache". https://nitro.build/guide/cache
[^5]: Nitro docs, "Deploy" (per-provider presets and route-rule support). https://nitro.build/deploy

## Tags

typescript, server-toolkit, web-server, serverless, edge-runtime, deployment, ssr, nuxt, vite, unjs
