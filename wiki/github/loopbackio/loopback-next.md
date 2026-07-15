# loopbackio/loopback-next

> A TypeScript-first Node.js API framework built around a dependency-injection container and OpenAPI-first controllers — the ground-up rewrite of the older StrongLoop LoopBack.

[GitHub repo](https://github.com/loopbackio/loopback-next) ·
[Official website](https://loopback.io) ·
[License: MIT](https://github.com/loopbackio/loopback-next/blob/master/LICENSE)

## Overview

LoopBack 4 (the `loopback-next` repository) is a Node.js framework for building REST APIs, designed around a dependency-injection (IoC) container and TypeScript decorators. It originated at StrongLoop, a startup acquired by IBM in 2015, and reached General Availability in October 2018[^1]. It is now maintained under the OpenJS Foundation with a technical steering committee led by long-time project architect Raymond Feng[^2].

LoopBack 4 is a complete rewrite of LoopBack 3, not an evolution of it. LoopBack 3 was a JavaScript, Express-based, convention-heavy framework where you described models in JSON and the framework generated dynamic endpoints. LoopBack 4 replaces that with an explicit, decorator-and-class programming model: you write `Controller`, `Repository`, and `Model` classes, wire them through a `Context` DI container, and the framework derives an OpenAPI 3 spec from your decorators. The tradeoff is the framework's defining tension — LoopBack 4 is far more type-safe and testable than its predecessor, but also considerably more verbose, and the migration path from the widely-deployed LoopBack 3 is nontrivial[^3]. LoopBack 3 reached end-of-life in December 2020[^1].

The project is still maintained (regular commits, active TSC), but its momentum has slowed relative to its 2018–2020 peak, and much of the broader Node.js TypeScript-API mindshare has moved to NestJS. It remains a reasonable choice where OpenAPI-first design and the legacy datasource/connector ecosystem are the deciding factors.

## Getting Started

```bash
npm create loopback@latest
# or, using the CLI directly:
npx -p @loopback/cli lb4 app
```

The Node.js baseline is a maintained LTS line (v20 or v22 as of 2026; the current/odd release is discouraged for production)[^2].

```ts
// src/controllers/ping.controller.ts
import {get} from '@loopback/rest';
import {inject} from '@loopback/core';
import {RestBindings, Request} from '@loopback/rest';

export class PingController {
  constructor(@inject(RestBindings.Http.REQUEST) private req: Request) {}

  @get('/ping', {
    responses: {
      '200': {description: 'Ping Response'},
    },
  })
  ping(): object {
    return {greeting: 'Hello from LoopBack', url: this.req.url};
  }
}
```

The decorators (`@get`, `@inject`) are the whole model: route metadata and the DI wiring are declared inline, and the OpenAPI spec is generated from them at boot.

## Architecture / How It Works

LoopBack 4 is built on its own IoC container, `@loopback/context`. Everything the app needs — controllers, repositories, datasources, config, request-scoped values — is a **Binding** in a `Context`, resolved lazily and injected via the `@inject` decorator[^4]. Contexts form a hierarchy (application → server → request), so bindings can be scoped per-request without a separate mechanism.

The request lifecycle runs through a **Sequence**: an ordered set of actions (`findRoute`, `parseParams`, `invoke`, `send`, `reject`). Overriding the default sequence is how you insert cross-cutting behavior — auth checks, logging, custom error shaping — rather than middleware stacking, though Express-style middleware is also supported at the REST layer.

The data layer is split in two:

- **Repository / Model** (`@loopback/repository`) — the typed, decorator-based API you write against. `@model`, `@property`, `@repository`, and relation decorators define entities and their CRUD repositories.
- **Juggler** — under that typed surface sits `loopback-datasource-juggler`, the same ORM/ODM abstraction inherited from LoopBack 3, plus its large family of `loopback-connector-*` packages (PostgreSQL, MySQL, MongoDB, REST, SOAP, and others). LoopBack 4 wraps juggler but does not replace it, so the repository layer's real capabilities and limits are juggler's.

The codebase is a Lerna monorepo of scoped packages (`@loopback/core`, `@loopback/rest`, `@loopback/repository`, `@loopback/authentication`, `@loopback/authorization`, `@loopback/boot`, and many more). **Components** bundle related bindings and extensions for reuse; **Interceptors** wrap controller method invocations (the DI-native equivalent of AOP-style around-advice).

## Production Notes

**Verbosity is the standing complaint.** A CRUD resource in LoopBack 4 spans a model, a repository, a controller, and datasource config — several files where LoopBack 3 needed one JSON file. The `lb4` CLI scaffolds most of this, but the generated code is yours to maintain, and the boilerplate is real. Teams evaluating LB4 should build one full resource before committing.

**Juggler is the ceiling of the data layer.** The ORM is adequate for straightforward CRUD and simple relations, but it is not a full-featured ORM. Complex joins, advanced SQL, and rich migration workflows are weak spots. `automigrate`/`autoupdate` (schema sync from models) are convenient in development but are not safe production migration tools — they can drop columns/data — so most teams pair LoopBack with an external migration tool for real schema management.

**Relations support has historically lagged.** Not every relation type reached parity with LoopBack 3, and nested/inclusion queries have edge cases. Verify the specific relations you need against the current docs rather than assuming full ORM behavior.

**LoopBack 3 → 4 migration is a rewrite, not an upgrade.** The programming models share almost no surface. There is an official migration guide and a `@loopback/booter-lb3app` compatibility shim to mount an existing LB3 app inside an LB4 app during transition, but plan for substantial work[^3].

**LTS.** As of the README there is no formal LTS release line for LoopBack 4; the project's own table lists LoopBack 4 EOL as "Apr 2028 (minimum)"[^1], and long-term-support commitments are still under discussion. Budget for tracking the current release rather than pinning to a supported-forever version.

## When to Use / When Not

**Use when:**
- OpenAPI-first (spec-driven) design is a hard requirement and you want the spec derived from typed code.
- You need one of juggler's many connectors (SOAP, legacy REST proxying, less-common databases) that other frameworks don't ship.
- You want strong DI and testability and are comfortable with an explicit, class-heavy structure.
- You're extending or maintaining an existing LoopBack estate.

**Avoid when:**
- You want minimal boilerplate for a small service — Express or Fastify are far leaner.
- You need a best-in-class ORM with strong migrations — pair Prisma/TypeORM with a lighter framework instead.
- You're picking a TypeScript DI framework for its ecosystem and hiring pool — NestJS is more active and more widely adopted in 2026.
- You're migrating a large LoopBack 3 app and hoped for an in-place upgrade.

## Alternatives

- nestjs/nest — the dominant TypeScript DI framework; larger ecosystem and community, similar decorator/DI model, more active development.
- expressjs/express — minimal, unopinionated; use when you want to assemble your own stack instead of a framework's.
- fastify/fastify — schema-based validation and high throughput; use when performance and a light footprint matter more than a full DI container.
- prisma/prisma — an ORM, not a framework; pair with any of the above when juggler's data layer is the weak point.
- LoopBack 3 (`strongloop/loopback`, EOL) — only relevant for existing deployments; not for new work.

## History

| Version | Date | Notes |
|---------|------|-------|
| LoopBack 2 | 2014-07 | StrongLoop era; Express-based, JS. EOL Apr 2019[^1]. |
| LoopBack 3 | 2016-12 | Widely deployed JS framework. EOL Dec 2020[^1]. |
| `loopback-next` repo created | 2017-01 | Ground-up TypeScript rewrite begins[^5]. |
| LoopBack 4 GA | 2018-10 | General Availability; DI container + OpenAPI-first controllers[^1]. |

## References

[^1]: LoopBack 4 README — "Status: General Availability" and version/EOL table. https://github.com/loopbackio/loopback-next#status-general-availability
[^2]: LoopBack 4 README — installation/Node.js requirements and Technical Steering Committee. https://github.com/loopbackio/loopback-next#team
[^3]: LoopBack docs — "Migrating from LoopBack 3". https://loopback.io/doc/en/lb4/migration-overview.html
[^4]: LoopBack docs — "Context" and dependency injection. https://loopback.io/doc/en/lb4/Context.html
[^5]: GitHub API — `loopbackio/loopback-next` repository `created_at` = 2017-01-09.

## Tags

typescript, nodejs, api-framework, rest, openapi, dependency-injection, orm, backend, ioc-container, strongloop
