# fastify/fastify

> Fast, low-overhead Node.js web framework whose performance story is built on compiling JSON Schema into serializers and validators.

[GitHub repo](https://github.com/fastify/fastify) ·
[Official website](https://www.fastify.dev) ·
[License: MIT](https://github.com/fastify/fastify/blob/main/LICENSE)

## Overview

Fastify is a Node.js HTTP framework created by Tomas Della Vedova and Matteo Collina, with a first stable release in 2018[^1]. It occupies the same niche as Express — routing, middleware-like hooks, request/reply objects — but its design thesis is that per-request overhead should be pushed to startup time. The two ideas that make it distinct are **schema-driven serialization** (declaring a response JSON Schema lets Fastify compile a purpose-built stringifier via `fast-json-stringify`, which is dramatically faster than `JSON.stringify` on hot paths) and an **encapsulation-based plugin system** (plugins get isolated contexts so decorators and hooks don't leak globally).

The framework is governed under the OpenJS Foundation as an At-Large project, and Matteo Collina is also a Node.js Technical Steering Committee member — so Fastify tends to track Node core closely and adopts new runtime features quickly. It is widely used as the HTTP layer under larger stacks: NestJS ships a Fastify adapter, and Platformatic (Collina's company) is built directly on it.

The defining tension is the schema requirement. Fastify's biggest wins — validation, serialization speed, and auto-generated OpenAPI — all depend on you writing JSON Schema for your routes. Teams that skip schemas get a framework that is still fast but gives up most of what differentiates it from Express, and they hit the serialization footgun (below) the first time a response field silently disappears.

## Getting Started

```sh
npm i fastify
```

```js
import Fastify from 'fastify'

const fastify = Fastify({ logger: true })

// A schema makes validation + fast serialization + OpenAPI possible.
fastify.get('/users/:id', {
  schema: {
    params: { type: 'object', properties: { id: { type: 'integer' } } },
    response: {
      200: {
        type: 'object',
        properties: { id: { type: 'integer' }, name: { type: 'string' } }
      }
    }
  }
}, async (request) => {
  return { id: request.params.id, name: 'Ada', secret: 'dropped' }
  // `secret` is NOT in the response schema → it is stripped from output.
})

await fastify.listen({ port: 3000, host: '0.0.0.0' })
```

`request.params.id` arrives as a coerced integer because the schema drives Ajv coercion; the `secret` field never reaches the client because `fast-json-stringify` only emits declared properties.

## Architecture / How It Works

Fastify is assembled from a small set of focused libraries, most maintained under the same org:

- **find-my-way** — a radix-tree HTTP router. Route lookup is O(path length), independent of the number of registered routes, which is where a large part of the throughput advantage over Express's linear route matching comes from.
- **Ajv** — JSON Schema validator. `body`, `querystring`, `params`, and `headers` schemas compile to validation functions once at boot.
- **fast-json-stringify** — turns a response schema into a specialized string-builder. This is the serialization speedup, and also the reason undeclared response fields vanish.
- **avvio** — the plugin bootstrapper. `register()` builds an async dependency graph; the server only starts listening after every plugin's `ready` state resolves.
- **Pino** — the logger. Fastify integrates it as a first-class, low-cost structured logger rather than reaching for `console`.

**Encapsulation** is the mental model that trips up newcomers. Each `register()`ed plugin gets a child context. Decorators (`fastify.decorate`), hooks, and route registrations added inside a plugin are visible to that plugin and its children, but **not to siblings or the parent**. To share something upward — a database connection, an auth decorator — you wrap the plugin with `fastify-plugin`, which tells avvio to skip creating a new encapsulation context and merge into the parent scope instead[^2].

**The request lifecycle** runs a fixed sequence of hooks: `onRequest` → `preParsing` → `preValidation` → `preHandler` → (handler) → `preSerialization` → `onSend` → `onResponse`, with `onError` and `onTimeout` as branches[^3]. Fastify calls these "hooks" rather than "middleware"; classic Express-style `(req, res, next)` middleware is supported only through the separate `@fastify/middie` or `@fastify/express` compatibility plugins, and is discouraged on hot paths.

The whole framework is built around this compile-at-boot / execute-fast split. It means startup does more work (schema compilation, plugin graph resolution) in exchange for lower steady-state per-request cost.

## Production Notes

**The serialization footgun.** Because `fast-json-stringify` emits only schema-declared properties, an incomplete or wrong response schema silently drops fields — no error, just missing data downstream. This is the single most common "why is my field gone" support thread. Either declare response schemas completely, or omit the response schema entirely (falling back to `JSON.stringify`) rather than half-declaring it. `additionalProperties: true` can be set when you deliberately want passthrough.

**Encapsulation surprises.** "My decorator is undefined in another plugin" almost always means you registered it inside an encapsulated plugin without `fastify-plugin`. Understand the scoping rules before building shared infrastructure; the error surface is confusing because there is no crash, just an absent property.

**Async vs callback reply.** A handler may either `return`/`throw` (async style) or call `reply.send()` (callback style) — but doing both, or returning a value after calling `reply.send`, causes double-send warnings and undefined behavior. Pick one style per handler.

**v4 → v5 upgrade (2024).** v5 raised the minimum to Node.js 20, moved to Ajv 8 defaults, and removed a batch of long-deprecated APIs[^4]. Most breakage is in plugins and in code relying on removed shorthand. The `4.x` branch still exists but v3 and earlier are end-of-life and receive no security fixes; HeroDevs sells commercial "never-ending support" patches for EOL versions[^5].

**Plugin ecosystem coupling.** Fastify's core is deliberately small; batteries (CORS, JWT, rate-limit, static, swagger, websocket) live in separate `@fastify/*` packages, each with its own version and its own Fastify-version compatibility range. A Fastify major bump usually means auditing every `@fastify/*` dependency for a matching major. Community plugins vary in maintenance quality.

**Benchmarks are synthetic.** The README's ~76k req/s figure is a hello-world overhead benchmark on specific hardware[^6]. It measures framework overhead, not your application, which will be dominated by your handlers, database, and serialization payload size. Treat it as "Fastify's floor is low," not as a throughput prediction.

**Containers.** `listen()` binds to `localhost` by default; inside Docker/Kubernetes you must pass `host: '0.0.0.0'` or the service is unreachable from outside the container.

## When to Use / When Not

**Use when:**
- You want low per-request overhead and are willing to write JSON Schema to get validation, fast serialization, and OpenAPI generation for free.
- You're building JSON APIs and want structured logging (Pino) and a clean hook lifecycle out of the box.
- You want an actively maintained, foundation-governed framework that tracks Node core closely.
- You're adopting NestJS or Platformatic and want the faster HTTP adapter underneath.

**Avoid when:**
- Your team won't write schemas — you lose most of Fastify's edge and inherit the serialization footgun.
- You have a large existing Express codebase leaning on `(req, res, next)` middleware; the compatibility shims work but you're fighting the grain.
- You're deploying to edge/multi-runtime targets (Cloudflare Workers, Deno, Bun-first) where a Web-Standards framework fits better.
- You need a minimal, zero-abstraction router with no lifecycle — Fastify's plugin/encapsulation model is more machinery than a tiny service needs.

## Alternatives

- expressjs/express — the incumbent; simpler mental model, vastly larger middleware ecosystem, slower and no built-in schema/serialization story. Use when ubiquity and hiring familiarity matter more than throughput.
- koajs/koa — minimal async-middleware core from the Express authors. Use when you want to assemble everything yourself around `async`/`await`.
- honojs/hono — Web-Standards, multi-runtime (Workers/Deno/Bun/Node). Use when you target the edge or non-Node runtimes.
- nestjs/nest — opinionated, DI-heavy application framework that can run *on* Fastify. Use when you want enterprise structure and decorators over a bare framework.
- hapijs/hapi — configuration-centric framework with built-in validation. Use when you prefer declarative config over Fastify's plugin encapsulation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-03 | First stable release; plugin system, schema-based serialization[^1]. |
| 2.0 | 2019-02 | Ajv upgrade, TypeScript type improvements. |
| 3.0 | 2020-07 | Reworked TypeScript types, Node 10+ baseline, async hook changes. |
| 4.0 | 2022-06 | Ajv 8, improved error handling, Node 14+ baseline[^6]. |
| 5.0 | 2024-09 | Node 20+ minimum, deprecated-API removal, Ajv defaults update[^4]. |

## References

[^1]: Fastify project history and authorship. https://www.fastify.dev/
[^2]: `fastify-plugin` — breaking encapsulation to share decorators/hooks. https://github.com/fastify/fastify-plugin
[^3]: Fastify docs, "Lifecycle" — hook execution order. https://www.fastify.dev/docs/latest/Reference/Lifecycle/
[^4]: Fastify docs, "V5 Migration Guide". https://www.fastify.dev/docs/latest/Guides/Migration-Guide-V5/
[^5]: Fastify README, "Support" — EOL policy and HeroDevs commercial support. https://github.com/fastify/fastify#support
[^6]: Fastify benchmarks (synthetic hello-world overhead). https://github.com/fastify/benchmarks

## Tags

nodejs, javascript, web-framework, http-server, rest-api, json-schema, plugin-architecture, performance, backend, openjs-foundation
