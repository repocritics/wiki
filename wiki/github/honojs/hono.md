# honojs/hono

> A small web framework built on Web Standards that runs the same code on Cloudflare Workers, Deno, Bun, Node.js, and AWS Lambda.

[GitHub repo](https://github.com/honojs/hono) ·
[Official website](https://hono.dev) ·
[License: MIT](https://github.com/honojs/hono/blob/main/LICENSE)

## Overview

Hono ("flame" in Japanese) is a server-side web framework created by Yusuke Wada, first published to npm in December 2021[^1]. Its thesis is that the JavaScript server ecosystem has fragmented across incompatible runtimes — Cloudflare Workers, Deno, Bun, Vercel Edge, Fastly Compute, AWS Lambda, and Node.js — and that a framework written against the WHATWG Web Standards (`Request`, `Response`, `fetch`, `URL`) rather than any one runtime's native API can run unmodified on all of them. In practice this is the framework's whole identity: the same `app.get('/', (c) => c.text('...'))` handler deploys to a Worker or a Lambda by swapping only the entry adapter.

The defining tension is that Web-Standards-first design pays off most at the edge and least on Node.js. Node did not implement the Fetch API (`Request`/`Response`) natively until relatively recently, so on Node, Hono runs through an adapter (`@hono/node-server`) that translates between Node's `http` streams and Web `Request`/`Response` objects. This adds a translation layer that edge runtimes don't need, and it is the source of most Node-specific footguns (streaming, file serving, and body-size behavior differ from the edge path).

Hono is small and has zero runtime dependencies — the `hono/tiny` preset is advertised under 12 kB[^2] — which matters because edge platforms bill and gate on bundle size and cold-start cost. It is best thought of as a routing-and-middleware core plus a large set of first-party helpers (validators, JWT, JSX renderer, an RPC client), not a batteries-included application framework in the Rails or Next.js sense. There is no ORM, no CLI-generated project structure, no opinions about data.

## Getting Started

```bash
npm create hono@latest my-app
# prompts: which template? (cloudflare-workers, bun, deno, nodejs, aws-lambda, ...)
cd my-app
npm install
npm run dev
```

```ts
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => c.text('Hono!'))

app.get('/users/:id', (c) => {
  const id = c.req.param('id')
  return c.json({ id })
})

export default app
```

On Node.js the entry point differs — you serve through the adapter rather than exporting the app:

```ts
import { serve } from '@hono/node-server'
import { Hono } from 'hono'

const app = new Hono()
app.get('/', (c) => c.text('Hono on Node'))

serve({ fetch: app.fetch, port: 3000 })
```

## Architecture / How It Works

The core is deliberately thin: a `Hono` instance holds a router and a middleware stack. A request becomes a `Context` object (`c`), which wraps the incoming `Request`, accumulates response state, and exposes helpers (`c.req`, `c.json`, `c.text`, `c.header`, `c.var`). Middleware is the standard onion model — each middleware receives `(c, next)`, calls `await next()` to descend, and can act on the response on the way back out.

The distinctive engineering is in the **routers**. Hono ships several interchangeable router implementations because routing is the hot path on every request and different runtimes reward different tradeoffs[^3]:

- **RegExpRouter** — compiles all registered routes into a small number of regular expressions so matching does not loop linearly over routes. Fastest for dispatch, but has higher registration cost and does not support every routing pattern.
- **TrieRouter** — a prefix-trie matcher that supports all patterns RegExpRouter cannot.
- **SmartRouter** — the default. It holds multiple routers and, on the first request, picks the fastest one that can handle the registered route set, caching that choice.
- **LinearRouter** — optimized for fast registration and near-zero startup, which matters on short-lived edge isolates where the app is constructed per cold start.
- **PatternRouter** — the smallest router by code size, for bundle-constrained deployments.

The routers were designed by Taku Amano[^1]. This multi-router design is why Hono can claim both fast startup and fast dispatch — it does not commit to one.

**Runtime portability** is achieved through adapters under `hono/*`: `hono/cloudflare-workers`, `hono/aws-lambda`, `hono/vercel`, `hono/deno`, `hono/bun`, plus the external `@hono/node-server`. Each adapter is responsible for turning the platform's native invocation into a Web `Request` and turning Hono's `Response` back into what the platform expects. Platform bindings (Workers KV, D1, environment) reach handlers through `c.env`, which is typed via generics.

**TypeScript inference** is a first-class concern. The RPC feature (`hono/client`) infers a fully typed client from the server app's route definitions, so a frontend can call `client.users[':id'].$get()` with end-to-end types and no code generation. The `@hono/zod-validator` (and `hono/validator`) middleware validates and narrows request input into the typed context. This inference is powerful but is also where compile times and cryptic type errors concentrate on large apps.

## Production Notes

**Node.js is a second-class-but-supported target.** The framework is designed edge-first; on Node you run through `@hono/node-server`, which is a translation shim. Streaming responses, `serveStatic`, and large request bodies behave differently than on Workers/Bun and have historically been where Node-specific bugs surface. If your only target is Node, a Node-native framework (Fastify, Express) will have fewer surprises; Hono on Node makes sense when you want the same codebase to also run on the edge.

**"Ultrafast" is router dispatch, not a guarantee of app throughput.** The RegExpRouter benchmarks measure route matching in isolation. Real request cost is dominated by your middleware, JSON serialization, validation (Zod is not cheap), and downstream I/O. Hono removes framework overhead from the hot path; it does not make your handlers fast.

**Bundle size discipline is on you.** The `hono/tiny` preset and per-router imports exist because edge platforms cap bundle size and charge for cold-start compile. Importing the full `Hono` from `'hono'` pulls SmartRouter and more; if you are size-constrained, import a specific router (`hono/quick`, `hono/tiny`) and only the middleware you use. Middleware and helpers are tree-shakeable but only if you import them granularly.

**RPC type inference can blow up compile times.** The end-to-end typed client is the headline TypeScript feature, but on apps with hundreds of routes the inferred types get large, `tsc` slows down, and error messages become long and hard to read. Splitting the app into multiple sub-apps and exporting narrower route types is the standard mitigation.

**`c.env` typing is manual.** Platform bindings (Cloudflare KV/D1/R2, secrets) are typed by generic parameters you declare yourself (`new Hono<{ Bindings: ... }>()`). Get this wrong and you get runtime `undefined` where TypeScript said the binding existed. Keep the `Bindings`/`Variables` types in sync with `wrangler.toml` by hand.

**Ecosystem middleware lives in a separate repo.** Built-in middleware ships in core, but many integrations (`@hono/zod-validator`, `@hono/swagger-ui`, `@hono/oauth-providers`, `@hono/node-server`) are separate packages in the `honojs/middleware` monorepo with independent versioning. Check each one's maintenance and peer-dependency ranges rather than assuming core's stability applies.

**Versioning.** Hono reached 4.0 in early 2024 and has iterated within v4 since. It follows semver and the project has kept breaking changes relatively contained, but it is a young framework compared to Express — pin versions and read release notes on minor bumps, since new helpers and occasional context-API adjustments land often.

## When to Use / When Not

**Use when:**
- You deploy to Cloudflare Workers, Deno Deploy, Bun, or another edge/serverless runtime and want minimal cold start and bundle size.
- You want one codebase that can move between runtimes without a rewrite.
- You want end-to-end TypeScript types from server routes to client calls without code generation.
- You're building an API or lightweight backend, not a full-stack rendered application.

**Avoid when:**
- You're Node-only and want the most battle-tested, largest-ecosystem option — Express and Fastify have more middleware, more Stack Overflow history, and no translation shim.
- You need a full-stack framework with data-layer opinions, file-based full-page routing, and a rendering story — reach for Next.js, Nuxt, or SvelteKit.
- Your team values a decade-stable API surface over an actively evolving one.
- Your bottleneck is I/O or database work, where framework dispatch speed is irrelevant.

## Alternatives

- expressjs/express — the Node default; far larger ecosystem and community memory, but Node-only, callback-era API, and slower routing.
- fastify/fastify — Node-focused, schema-based validation and serialization, strong performance; choose it when you're Node-only and want maturity plus speed.
- oakserver/oak — Deno-native Koa-style framework; use it when you're committed to Deno and prefer its idioms over Web-Standards portability.
- elysiajs/elysia — Bun-first framework with heavy TypeScript inference and high benchmarks; use it when Bun is your only target and you want maximal type ergonomics.
- itty-router — sub-kilobyte Workers router with almost no features; use it when you want routing and nothing else in the tightest possible bundle.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-12 | First npm publish; Cloudflare Workers focus[^1]. |
| 2.0 | 2022 | Router rework, expanded middleware. |
| 3.0 | 2023 | Multi-runtime adapters, RPC/typed client, JSX renderer maturing. |
| 4.0 | 2024-02 | Major release; API cleanups, `hono/tiny` and preset split[^4]. |

## References

[^1]: Hono repository and authorship (Yusuke Wada; routers by Taku Amano). https://github.com/honojs/hono
[^2]: Hono documentation, "Features" — zero dependencies, `hono/tiny` preset size. https://hono.dev/docs/concepts/motivation
[^3]: Hono documentation, "Routers" — RegExpRouter, SmartRouter, TrieRouter, LinearRouter, PatternRouter. https://hono.dev/docs/api/routing
[^4]: Hono blog, "Hono v4.0.0" — 2024-02. https://hono.dev/blog

## Tags

typescript, web-framework, edge-computing, cloudflare-workers, serverless, http-router, middleware, web-standards, deno, bun, api
