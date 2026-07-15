# hapijs/hapi

> Configuration-centric HTTP framework for Node.js, built at Walmart for high-traffic APIs and now community-governed.

[GitHub repo](https://github.com/hapijs/hapi) ·
[Official website](https://hapi.dev) ·
[License: BSD-3-Clause](https://github.com/hapijs/hapi/blob/master/LICENSE.md)

## Overview

hapi (HTTP API) is a server framework for Node.js first written in 2011 by Eran Hammer at Walmart Labs to survive Black Friday traffic on a large, multi-team codebase[^1]. Its founding premise is the opposite of Express: rather than a chain of `(req, res, next)` middleware, hapi is configuration-driven. A route is a plain object — method, path, and an `options` block declaring auth, validation, caching, payload rules — and the framework enforces that configuration rather than letting handlers mutate a shared request pipeline. The stated goal was to make large teams' intent auditable from the route table alone.

That philosophy is hapi's defining tension. The structure pays off on big, long-lived API services where explicit contracts and a strict request lifecycle prevent middleware-ordering bugs; it feels heavy on small services where Express or Fastify get you to "hello world" faster. hapi also ships as a galaxy of small first-party modules under the `@hapi/*` npm scope — boom (errors), hoek (utilities), catbox (caching), podium (events), and roughly a dozen more — which the framework composes internally[^2]. You rarely import them directly, but they define hapi's dependency surface and its release cadence.

As of mid-2026 the repository sits around 14.8k stars with ~1.35k forks and a last push in May 2026 — mature and maintained, but past its mindshare peak. Express and Fastify dominate new-project attention; hapi's following is smaller, older, and concentrated in enterprises that adopted it in the 2014–2019 window and value its stability over novelty. Governance moved from a single author to a Technical Steering Committee after 2020[^3].

## Getting Started

```bash
npm install @hapi/hapi
```

```js
const Hapi = require('@hapi/hapi');

const init = async () => {
  const server = Hapi.server({ port: 3000, host: 'localhost' });

  server.route({
    method: 'GET',
    path: '/hello/{name}',
    handler: (request, h) => {
      return { message: `Hello ${request.params.name}` };
    }
  });

  await server.start();
  console.log('Server running on %s', server.info.uri);
};

process.on('unhandledRejection', (err) => { console.error(err); process.exit(1); });
init();
```

Validation is declared on the route, not inside the handler — the request never reaches your code if it fails:

```js
const Joi = require('joi');

server.route({
  method: 'POST',
  path: '/users',
  options: {
    validate: {
      payload: Joi.object({ name: Joi.string().min(1).required() })
    }
  },
  handler: (request, h) => h.response(request.payload).code(201)
});
```

## Architecture / How It Works

There is no middleware stack. Every request runs a fixed **request lifecycle** with named extension points — `onRequest`, `onPreAuth`, `onCredentials`, `onPostAuth`, `onPreHandler`, `onPostHandler`, `onPreResponse` — and exactly one handler per route[^4]. You attach behavior to a specific lifecycle step rather than ordering an array of functions, which removes the class of bugs where middleware runs in the wrong sequence. The tradeoff is a steeper mental model: you must learn the lifecycle to know where anything belongs.

**Everything is a plugin.** Features register through `server.register()`, and even application routes are typically packaged as plugins. Plugins get a scoped `server` object, can expose services via `server.expose()`, declare dependencies, and are isolated enough to be composed across a large app. Static files (inert), templating (vision), third-party OAuth (bell), and WebSockets (nes) are all first-party plugins rather than core.

The handler contract is async-native. Since hapi 17 there are no callbacks: handlers are `async` functions that **return** a value or an `h.response()` toolkit object; the return value becomes the response and thrown errors (usually `Boom` objects) become HTTP status codes[^5]. Validation leans on Joi — recent hapi versions vendor a trimmed fork as `@hapi/validate` internally while most application code still uses the standalone `joi` package.

Internally hapi is assembled from its own modules: `call` (the router), `subtext` (payload parsing), `statehood` (cookies), `catbox` (pluggable server-side caching with memory/Redis/Memcached adapters), `heavy` (load-shedding under memory/event-loop pressure), and `shot` (in-process request injection, which powers testing without opening a socket). This means a hapi upgrade is really a coordinated bump across a dozen packages, and the framework's behavior is stable precisely because those packages change slowly.

## Production Notes

- **Throughput is not hapi's headline.** The lifecycle, per-request validation, and object-heavy config cost measurable overhead versus Fastify's schema-compiled fast path. For most I/O-bound APIs this is irrelevant, but if raw requests/sec is your bottleneck, hapi is the wrong tool and no config flag closes the gap.
- **`server.inject()` is the testing superpower.** Because `shot` dispatches requests through the full lifecycle in-process, integration tests need no live port and run fast and deterministically. Teams that adopt this get very high confidence; teams that ignore it miss hapi's best feature.
- **Node version floor moves deliberately.** hapi 21 requires Node >= 14.15.0[^6]. The project tracks Node LTS and drops EOL versions on major bumps; check the version-status page before pinning, since older majors stop receiving fixes.
- **Load protection is built in but off by default.** `heavy` can return 503s when the event-loop delay or heap usage crosses configured thresholds — useful for shedding load before a crash, but you must set `load.maxEventLoopDelay` / `maxHeapUsedBytes` explicitly.
- **Caching requires an adapter decision.** catbox defaults to in-memory, which does not survive multi-instance deploys; production caching means wiring a Redis or Memcached catbox adapter and reasoning about its invalidation yourself.
- **Ecosystem gravity.** Because auth, validation, and file serving are all `@hapi/*` plugins, you are buying into that scope's release schedule. Third-party plugin coverage is thinner than Express's, and some community plugins lagged for months after the async-only hapi 17 rewrite.

## When to Use / When Not

**Use when:**
- You are building a large or multi-team API service where explicit, auditable route configuration beats terse middleware.
- You want first-party, coherent answers for auth, validation, caching, and testing rather than assembling middleware yourself.
- Long-term stability and a slow, deliberate breaking-change cadence matter more than being on the newest thing.

**Avoid when:**
- You want minimalism, the largest plugin ecosystem, or the most tutorials — Express still wins on mindshare.
- Peak requests/sec is the goal — Fastify is faster by design.
- You want TypeScript-first decorators and dependency injection out of the box — that is Nest's model, not hapi's.

## Alternatives

- expressjs/express — minimal middleware framework with the largest ecosystem and mindshare; use when you want maximum flexibility and community plugins over enforced structure.
- fastify/fastify — schema-first framework with hapi-like plugin encapsulation but much higher throughput; use when you want structure *and* speed with current momentum.
- nestjs/nest — opinionated, TypeScript-first with DI and decorators; use when you want an enterprise, Angular-style architecture on top of Express or Fastify.
- koajs/koa — lightweight async middleware from the Express authors; use when you want modern async/await middleware with almost no framework.
- restify/node-restify — REST-only framework with built-in observability; use when you are strictly building machine-facing REST APIs.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-08 | Created at Walmart Labs by Eran Hammer for Black Friday-scale APIs[^1]. |
| 8–16 | 2014–2017 | Callback-era maturity; adoption peak across enterprise Node shops. |
| 17.0 | 2017-11 | Full async/await rewrite; callbacks and the `reply()` interface removed[^5]. |
| 18–19 | 2019 | Modules consolidated under the `@hapi/*` npm scope[^2]. |
| 20.0 | 2020 | Governance transition; project moved to a Technical Steering Committee[^3]. |
| 21.0 | 2022 | Node >= 14.15.0 floor; current major line[^6]. |
| 21.4.9 | 2026-05 | Latest published release at time of writing. |

## References

[^1]: hapi origin and design goals — hapi.dev. https://hapi.dev/
[^2]: hapi module ecosystem under the `@hapi` scope (boom, hoek, catbox, joi/validate, inert, vision, bell, nes). https://hapi.dev/family/
[^3]: hapi project policies and Technical Steering Committee governance. https://hapi.dev/policies/
[^4]: Request lifecycle and extension points. https://hapi.dev/api/#request-lifecycle
[^5]: hapi 17 async/await migration (removal of callbacks). https://hapi.dev/resources/status/#hapi
[^6]: hapi `package.json` engines field, `node >= 14.15.0` (repo `master`, v21.4.9). https://github.com/hapijs/hapi/blob/master/package.json

## Tags

javascript, nodejs, web-framework, http-server, backend, rest-api, api, plugin-architecture, validation, enterprise
