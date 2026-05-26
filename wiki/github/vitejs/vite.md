# vitejs/vite

> Next-generation frontend tooling — native ESM dev server, Rollup/Rolldown production bundler.

[GitHub repo](https://github.com/vitejs/vite) ·
[Official website](https://vitejs.dev) ·
[License: MIT](https://github.com/vitejs/vite/blob/main/LICENSE)

## Overview

Vite is a JavaScript build tool created by Evan You in 2020[^1], originally as a faster dev server for Vue 3 and quickly generalized to a framework-agnostic toolchain. Its core trick is that dev mode does not bundle: it serves source files directly to the browser over native ES modules, and dependencies are pre-bundled once with esbuild. Production builds use Rollup (and are migrating to Rolldown[^2], a Rust port).

Vite has become the default bundler for almost every non-Next.js modern web framework: Nuxt, SvelteKit, SolidStart, Remix (pre-merge), Astro, Qwik, and Rakkas. It is the substrate the React-and-friends ecosystem moved to once Webpack became the friction point. As of 2026 Rolldown is in active integration, and the Vite → Rolldown migration is the single largest open architectural question in the project.

## Getting Started

```bash
npm create vite@latest my-app
# prompts: framework? (vanilla, vue, react, preact, lit, svelte, solid, qwik)
# variant? (TS / JS, SWC, etc.)
cd my-app
npm install
npm run dev    # http://localhost:5173
npm run build  # static output in dist/
```

```ts
// vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: { port: 3000 },
  build: { sourcemap: true, target: "es2022" },
});
```

For library mode:

```ts
export default defineConfig({
  build: {
    lib: { entry: "src/index.ts", formats: ["es", "cjs"] },
    rollupOptions: { external: ["react", "react-dom"] },
  },
});
```

## Architecture / How It Works

Vite has two execution modes that share configuration and plugin protocols but otherwise have different runtimes:

1. **Dev server** — A Node.js (or, optionally, Deno/Bun) process serves source files over HTTP. Imports are rewritten on the fly: `import "./foo"` becomes `import "/src/foo.ts?v=hash"`, source files are transformed lazily on request. HMR is implemented over a WebSocket; each module declares its accept boundary via `import.meta.hot.accept()`. Dependencies in `node_modules` are pre-bundled by esbuild on first run and cached in `node_modules/.vite`.

2. **Production build** — Currently Rollup with a plugin layer that adapts Rollup plugins to Vite's plugin protocol (which is a strict superset of Rollup's). Rolldown — a Rust Rollup-compatible bundler[^2] — is shipping as an opt-in alternative in Vite 6 and is the planned default in a future major.

The plugin system is the unifying abstraction. A Vite plugin is a Rollup plugin (`resolveId`, `load`, `transform`) with optional Vite-only hooks (`configResolved`, `configureServer`, `transformIndexHtml`, `handleHotUpdate`). The same plugin runs in both dev and build modes, with the framework deciding which hooks to fire when[^3].

Vite 5+ introduced **Environment API** (formalized in 6) — a mechanism for multiple parallel module graphs in a single dev server, enabling SSR, RSC, and worker bundles to share a process without separate config files[^4]. This is the substrate frameworks like Nuxt, SvelteKit, and SolidStart use to express server + client + edge builds.

## Production Notes

**Pre-bundling is invalidation-sensitive.** Vite caches optimized dependencies in `node_modules/.vite` keyed by a lockfile hash. When the cache stales but the hash doesn't change (e.g., a monorepo `pnpm` patch), you get cryptic "outdated optimize" errors. `rm -rf node_modules/.vite` is the standard first step in any dev-server debugging session.

**Dev vs. production divergence.** Dev serves native ESM; production produces Rollup-bundled chunks. A surprising amount of CSS, environment variable, dynamic-import-with-template-literal, and side-effect-tree-shaking behavior differs between the two. Always test production builds (`npm run build && npm run preview`) before assuming dev behavior is representative.

**SSR.** Vite has a built-in SSR transform pipeline but does not provide an SSR runtime — you bring your own server (Express, Hono, Fastify, Cloudflare Worker) and use Vite's `ssrLoadModule` in dev and the SSR build output in production. Frameworks (SvelteKit, Nuxt, SolidStart) abstract this entirely.

**Plugin ecosystem.** The Vite plugin registry is large but uneven. Plugins written before the Environment API exists may not be SSR-safe. Rollup plugins largely work but may rely on Rollup-only hooks; the Vite docs maintain a compatibility note[^3].

**Build performance.** Rollup builds get slow on large apps (10k+ modules). Rolldown adoption is the path forward; teams migrating early should expect rough edges on niche plugins.

**Module resolution.** Vite uses its own resolver, not Node's, and supports `exports`/`imports` fields, `package.json#type`, and conditional exports. Older packages that ship CJS-only or have malformed `exports` maps occasionally need `optimizeDeps.include` or `ssr.noExternal` overrides.

## When to Use / When Not

**Use when:**
- You want fast dev startup and HMR on a SPA or library.
- You're not using Next.js / Remix and want a sane default build toolchain.
- You're building a meta-framework: Vite is the lowest-friction substrate for one.
- You need Rollup output without Webpack's complexity.

**Avoid when:**
- You need RSC with no glue code: Next.js still has the tightest RSC integration; Vite's RSC story is framework-mediated.
- You need a long-lived stable plugin ecosystem with no migration churn: the Rolldown migration will produce churn in 2026–2027.
- You're shipping a Webpack-coupled enterprise app whose plugin investments don't have Vite equivalents.

## Alternatives

- [oven-sh/bun](../oven-sh/bun.md) — built-in bundler; faster cold-builds, smaller plugin ecosystem.
- esbuild (not yet in this wiki) — Vite uses esbuild for pre-bundling; can be used standalone.
- Rspack (not yet in this wiki) — Rust port of Webpack; closer migration path from Webpack projects.
- Parcel (not yet in this wiki) — zero-config bundler; less plugin surface.
- Webpack (not yet in this wiki) — the historical baseline; still dominates legacy + enterprise.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2021-02-16 | First stable; Vue 3 focus[^1]. |
| 2.0 | 2021-08 | Framework-agnostic, multi-framework templates. |
| 3.0 | 2022-07 | ESM-first node config, dev server perf. |
| 4.0 | 2022-12 | Rollup 3, faster HMR. |
| 5.0 | 2023-11 | Node 18+, Rollup 4, sub-path imports. |
| 6.0 | 2024-11 | Environment API stable[^4]; Rolldown opt-in begins. |

## References

[^1]: Evan You, "Announcing Vite 1.0" — 2021-02-16. https://vuejs.org/blog/announcing-vite.html
[^2]: Rolldown project. https://rolldown.rs/
[^3]: Vite docs, "Plugin API". https://vitejs.dev/guide/api-plugin.html
[^4]: Vite docs, "Environment API". https://vitejs.dev/guide/api-environment.html
