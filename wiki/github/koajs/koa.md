# koajs/koa

> Minimal HTTP middleware for Node.js from the Express authors, rebuilt around async/await and an onion-model middleware stack.

[GitHub repo](https://github.com/koajs/koa) ·
[Official website](https://koajs.com) ·
[License: MIT](https://github.com/koajs/koa/blob/master/LICENSE)

## Overview

Koa is a small HTTP framework for Node.js created in 2013 by TJ Holowaychuk and the team behind Express, intended as a spiritual successor that shed Express's callback-era baggage[^1]. Its guiding idea is deliberate smallness: the core is roughly 570 lines of source and ships with no router, no body parser, no templating, and no bundled middleware. What it provides is a request/response abstraction (`Context`) and a middleware-composition mechanism. Everything else is assembled from the `@koa/*` and community packages.

The defining feature is the middleware model. Each middleware receives `(ctx, next)` and `await`s `next()` to yield to the next middleware downstream, then resumes to run code upstream after the rest of the stack has completed. This "onion" flow — enter each layer inward, unwind outward — makes cross-cutting concerns like timing, error wrapping, and response mutation natural to express, and it was the whole point of Koa: replacing Express's linear callback chains with a stack that reads top-to-bottom and unwinds bottom-to-top[^2].

The central tension is that Koa's minimalism is both the reason to choose it and the reason many teams don't. You get a clean, modern async core with no magic, but you are responsible for selecting and wiring routing, body parsing, security headers, and validation yourself — decisions Express makes for you and Fastify makes faster. Koa optimizes for control and correctness over batteries-included convenience.

## Getting Started

Koa v3 requires Node.js v18.0.0 or higher[^3].

```bash
npm install koa
```

```js
const Koa = require('koa');
const app = new Koa();

// Onion-model middleware: timing wraps the whole downstream stack
app.use(async (ctx, next) => {
  const start = Date.now();
  await next();                       // yield to downstream middleware
  ctx.set('X-Response-Time', `${Date.now() - start}ms`);
});

app.use(async (ctx) => {
  ctx.body = 'Hello Koa';            // ctx delegates to the underlying res
});

app.listen(3000);
```

Routing is not built in. The common pairing is `@koa/router`, and a body parser (`@koa/bodyparser` or `koa-body`) is added for POST/PUT handling.

## Architecture / How It Works

Koa's core is four objects: `Application`, `Context`, `Request`, and `Response`. `new Koa()` creates the application; each incoming request produces a fresh `Context` (`ctx`) that composes a Koa `Request` and `Response`.

The most important design decision is delegation rather than extension. Koa does not monkey-patch Node's `IncomingMessage`/`ServerResponse`. Instead `ctx.request` and `ctx.response` are Koa-owned objects whose accessors delegate down to the raw Node objects, still reachable as `ctx.req` and `ctx.res`. This avoids the whole class of conflicts Express created by mutating the native prototypes, and it means multiple middleware manipulating the response interact through Koa's abstraction instead of through Node internals[^2]. Frequently used members are also aliased directly onto `ctx` (e.g. `ctx.body`, `ctx.status`, `ctx.type`, `ctx.accepts`) as shortcuts to the `request`/`response` equivalents.

Middleware composition is the other half. `app.use()` pushes a function onto an array; on each request Koa reduces that array into a single promise chain where each `next()` returns a promise for the remainder of the stack. Because the whole chain is promise-based, `try/catch` around `await next()` catches errors thrown anywhere downstream — this is how centralized error handling and response-time measurement are written as ordinary async code.

`ctx.body` accepts strings, buffers, streams, objects (serialized to JSON), or `null`. Koa infers `Content-Type` and `Content-Length`, and for streams it pipes and manages cleanup. Setting a status and letting `ctx.throw(code)` / `ctx.assert()` raise HTTP errors are built into the context.

The middleware signature is version-load-bearing. Koa v1 used generator functions with `yield next` (relying on the `co` library); v2 moved to `async (ctx, next)` with `await next()`; v3 removed the v1-style signature and its compatibility shims entirely. Mixing eras requires wrappers like `koa-convert`, and this is the single biggest source of migration friction across the ecosystem.

## Production Notes

- **You own the security surface.** Koa ships no helmet-style headers, no CORS, no rate limiting, no CSRF. These are separate packages (`koa-helmet`, `@koa/cors`, etc.) that you must add deliberately. A "default" Koa app is not hardened.
- **Body parsing is not included.** Requests are not parsed until you add a body parser, and misconfigured size limits or missing parsers are a common cause of hung or silently-empty request handlers.
- **Error handling must be explicit.** An uncaught error in middleware emits an `'error'` event on the app and returns a 500. Register an `app.on('error', ...)` listener and a top-level try/catch middleware; without them, failures are opaque in production.
- **Return values from middleware are ignored — you must `await next()`.** Forgetting to await `next()` breaks the onion unwinding: upstream code runs before downstream finishes, producing responses sent too early or headers set after the body. This is the classic Koa footgun.
- **Streaming bodies bypass some middleware assumptions.** Setting `ctx.body` to a stream defers sending; middleware that inspects the body upstream may see a stream object rather than serialized content.
- **Migration cost is real.** The v1→v2 (generators→async) and v2→v3 (removal of legacy signature) transitions each break middleware that depended on the older contract. Audit third-party `koa-*` packages for maintenance and v3 compatibility before upgrading; many popular middleware are community-maintained and lag core releases.
- **Ecosystem is smaller than Express.** Fewer tutorials, fewer Stack Overflow answers, and a middleware catalog that, while high quality, is thinner. This matters most for teams expecting a package to already exist for a given need.

## When to Use / When Not

**Use when:**
- You want a minimal, modern async core and are comfortable selecting your own router, parser, and middleware.
- Async/await ergonomics and clean centralized error handling matter more than batteries-included convenience.
- You are building APIs or middleware-heavy services where the onion model maps cleanly to your cross-cutting concerns.

**Avoid when:**
- You want a framework that decides routing, parsing, and structure for you out of the box (use Express or a full framework).
- Raw throughput is a top priority and you want schema-based serialization and validation built in (use Fastify).
- Your team is large or junior and benefits from the deepest possible pool of examples and third-party middleware.

## Alternatives

- expressjs/express — the older sibling from the same authors; larger ecosystem, callback/linear middleware, built-in routing. Use it when you want defaults and the widest community.
- fastify/fastify — schema-driven, higher throughput, first-class TypeScript and plugin encapsulation. Use it when performance and validation are priorities.
- honojs/hono — small, Web-Standards-based, runs on Node, Bun, Deno, Cloudflare Workers. Use it when you target edge/multi-runtime deployment.
- nestjs/nest — opinionated, DI-based application framework (can run on Express or Fastify). Use it when you want enforced architecture at scale.
- hapijs/hapi — configuration-centric framework with built-in validation and auth. Use it when you prefer configuration over middleware assembly.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial commit | 2013-07-20 | Repository created by the Express authors[^1]. |
| 1.0 | 2015 | First stable release; generator-based middleware via `co` (`yield next`). |
| 2.0 | 2017 | Rewritten around `async`/`await`; `(ctx, next)` signature became canonical[^4]. |
| 3.0 | 2025 | Node 18+ baseline; v1-style middleware signature and legacy shims removed[^5]. |

## References

[^1]: koajs/koa repository, created 2013-07-20. https://github.com/koajs/koa
[^2]: Koa README — middleware stack, context delegation to Node's req/res. https://github.com/koajs/koa/blob/master/Readme.md
[^3]: Koa README — "Koa requires node v18.0.0 or higher." https://github.com/koajs/koa/blob/master/Readme.md
[^4]: Migration Guide from v1.x to v2.x. https://github.com/koajs/koa/blob/master/docs/migration-v1-to-v2.md
[^5]: Migration Guide from v2.x to v3.x. https://github.com/koajs/koa/blob/master/docs/migration-v2-to-v3.md

## Tags

javascript, nodejs, http, middleware, web-framework, backend, async-await, api, minimalist, koa
