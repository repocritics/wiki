# expressjs/express

> The minimal, unopinionated HTTP framework that became Node.js's default web layer — and the compatibility baseline everything else measures against.

[GitHub repo](https://github.com/expressjs/express) ·
[Official website](https://expressjs.com) ·
[License: MIT](https://github.com/expressjs/express/blob/master/LICENSE)

## Overview

Express is a thin routing and middleware layer over Node.js's built-in `http` module, first released by TJ Holowaychuk in 2010[^1]. It grew out of and eventually absorbed Connect, the earlier middleware framework, and for most of the following decade it was the near-universal answer to "how do I write an HTTP server in Node." Its API — `app.get(path, handler)`, `(req, res, next)` middleware, `res.send()` — is so widely copied that dozens of competing frameworks advertise themselves as "Express-compatible."

The defining characteristic is deliberate minimalism. Express does not ship an ORM, a template engine, a validation layer, or an opinion about project structure. It gives you routing, a middleware pipeline, and a handful of HTTP convenience helpers, and leaves everything else to userland packages. This is simultaneously its greatest strength (you assemble exactly what you need, nothing is hidden) and its central tension: a real Express application is defined mostly by the middleware you bolt on, and the framework offers little guidance on doing that safely or consistently.

The project's other defining trait is its glacial-then-sudden release cadence. Express 4.0 shipped in 2014 and remained the default for over a decade. Express 5.0 was in development for roughly ten years, spending years in alpha/beta limbo before finally reaching a stable release in 2024 and becoming the npm `latest` default in 2025[^2]. That gap shaped the ecosystem: a generation of tutorials, middleware, and Stack Overflow answers assume Express 4 semantics.

## Getting Started

```bash
npm install express   # Node.js 18+ required for Express 5
```

```js
import express from 'express'

const app = express()

app.use(express.json())                      // body parsing is built in as of v5

app.get('/users/:id', (req, res) => {
  res.json({ id: req.params.id })
})

// Error-handling middleware — identified by its 4-arg signature
app.use((err, req, res, next) => {
  console.error(err)
  res.status(500).json({ error: 'internal' })
})

app.listen(3000, () => console.log('listening on :3000'))
```

## Architecture / How It Works

Express is built around a single linear abstraction: the **middleware stack**. An application holds an ordered array of layers, each a `(req, res, next)` function paired with an optional path/method matcher. On each request, Express walks the stack in order; every layer either responds, or calls `next()` to pass control onward, or calls `next(err)` to jump to the error-handling middleware (functions with the four-argument `(err, req, res, next)` signature). Routers are themselves middleware, so the stack is really a tree of nested stacks. This model is small enough to hold in your head, and that legibility is a large part of why Express won.

Path matching is delegated to the `path-to-regexp` library, which compiles route patterns like `/users/:id` into regular expressions. Express 5 upgraded to a newer major version of `path-to-regexp` that removed several long-standing pattern features (notably unnamed regex groups and the bare `*` wildcard, now requiring a named `*splat`)[^3]. This is one of the more disruptive v4→v5 breaks because route strings are stringly-typed and fail at startup, not compile time.

`req` and `res` are not Express types — they are Node's native `http.IncomingMessage` and `http.ServerResponse`, monkey-patched with extra methods (`res.json`, `res.redirect`, `req.params`, and so on) via prototype extension. This keeps Express interoperable with the raw Node ecosystem but means the framework's surface leaks Node's HTTP quirks directly to your handlers.

Historically Express bundled functionality via the Connect-era practice of separate packages (`body-parser`, `serve-static`, `cookie-parser`). Express 4 externalized most of these; Express 5 re-internalized the common ones (`express.json()`, `express.urlencoded()`, `express.static()`) so a basic app needs fewer direct dependencies.

## Production Notes

**Async errors were a footgun until v5.** In Express 4, a rejected promise or an error thrown inside an `async` route handler is *not* caught by the framework — it becomes an unhandled rejection and the request hangs until timeout. Teams worked around this with wrappers like `express-async-handler` or `express-async-errors`. Express 5 fixes this: rejected promises returned from handlers are automatically forwarded to error middleware[^2]. If you are still on v4, every async handler needs a try/catch or a wrapper, and this is one of the most common sources of silent production hangs.

**Security is your responsibility, not the framework's.** A bare Express app sets no security headers, has no rate limiting, no CSRF protection, and permissive defaults. The de facto baseline is `helmet` for headers plus explicit `cors` configuration; skipping them is a frequent finding in security reviews. `express.json()` should be given a `limit` — the default body size is small but the parser will still buffer request bodies into memory.

**Performance is "fine," not exceptional.** Express's routing and prototype patching add measurable overhead versus lighter frameworks like Fastify or raw `http`. For most I/O-bound workloads the framework is not the bottleneck, but on high-throughput JSON APIs the difference (often cited in the ~2x range in Fastify's own benchmarks[^4]) can matter. Do not choose Express for raw throughput; choose it for ecosystem and familiarity.

**The v4→v5 migration is real work.** Beyond the routing pattern changes, v5 dropped a number of deprecated method signatures, changed `res.status()` handling of invalid codes, and requires Node 18+. Because so much middleware was written against v4, audit your dependency tree for compatibility before upgrading a large app. Many projects remained on v4 for years specifically because v5 was perpetually unreleased; that inertia is now the migration debt.

**Maintenance history matters for risk assessment.** After the original author stepped back, Express spent several years under thin maintenance — for a long stretch Douglas Wilson carried much of the load. In 2024 the project restructured under an OpenJS Foundation technical committee and a formal triage team, and shipped v5 and a security-focused release cadence[^5]. It is healthier now than in the 2016–2020 period, but its recent history is a caution against assuming a ubiquitous project is well-resourced.

## When to Use / When Not

**Use when:**
- You want the most widely understood Node HTTP model, with an answer for every question already on the internet.
- You need maximum middleware compatibility — the largest ecosystem of ready-made layers targets Express.
- Your workload is I/O-bound (typical CRUD/REST/BFF services) where framework overhead is negligible.
- You value a small, legible core you can reason about over batteries-included convenience.

**Avoid when:**
- Raw request throughput or built-in schema validation is a priority — Fastify is faster and validates by default.
- You want strong typing end-to-end — Express's TypeScript story is retrofitted (`@types/express`) and loose; NestJS or tRPC give stronger guarantees.
- You want opinions and structure out of the box — Express deliberately provides neither, and greenfield teams often reinvent the same scaffolding.
- You're targeting non-Node runtimes (Deno, Bun edge, Cloudflare Workers) — Hono is built for that portability.

## Alternatives

- fastify/fastify — use when throughput and built-in JSON-schema validation matter; similar middleware ergonomics, faster core.
- honojs/hono — use when you need one API across Node, Bun, Deno, and edge/Workers runtimes.
- koajs/koa — use when you want the same authors' async/await-native successor with a smaller, middleware-only core.
- nestjs/nest — use when you want an opinionated, DI-based, TypeScript-first structure (and it can run on Express under the hood).
- expressjs/express (v4) — use the older major only when a critical dependency has no v5-compatible release yet.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2010 | Initial release by TJ Holowaychuk, built on Connect[^1]. |
| 3.0 | 2012 | Middleware restructuring; Connect dependency reworked. |
| 4.0 | 2014-04 | Connect middleware externalized into separate packages; router rewrite. |
| 5.0.0 | 2024 | Stable after ~10 years in development; async error forwarding, `path-to-regexp` upgrade, Node 18+, body parsers re-internalized[^2]. |
| 5.1.0 | 2025 | Promoted to the npm `latest` default tag[^2]. |

## References

[^1]: Express project history and original authorship (TJ Holowaychuk). https://github.com/expressjs/express/graphs/contributors
[^2]: "Express 5.0 released" and v5.1 default-tag announcement, Express blog. https://expressjs.com/2024/10/15/v5-release.html
[^3]: Express 5 migration guide — routing / `path-to-regexp` changes. https://expressjs.com/en/guide/migrating-5.html
[^4]: Fastify benchmarks (framework's own comparison figures). https://fastify.dev/benchmarks/
[^5]: OpenJS Foundation governance and technical committee for Express. https://github.com/expressjs/discussions/blob/HEAD/docs/GOVERNANCE.md

## Tags

javascript, nodejs, web-framework, http-server, middleware, rest-api, backend, routing, minimalist, mit-license
