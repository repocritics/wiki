# feathersjs/feathers

> A service-and-hooks framework that gives you REST plus automatic real-time events over any database, in TypeScript or JavaScript.

[GitHub repo](https://github.com/feathersjs/feathers) ·
[Official website](https://feathersjs.com) ·
[License: MIT](https://github.com/feathersjs/feathers/blob/dove/LICENSE)

## Overview

Feathers is a Node.js framework for building web APIs and real-time applications, first released as an open-source project by David Luecke and Bitovi around 2011–2012[^1]. Its defining idea is the **service**: a small object implementing a fixed set of methods (`find`, `get`, `create`, `update`, `patch`, `remove`). Feathers wraps each service so the same code is reachable over REST and over websockets, and — this is the part that distinguishes it — any mutation automatically emits a real-time event (`created`, `patched`, `removed`, …) that is pushed to subscribed clients. You write CRUD once; you get a REST API, a live query layer, and a database-agnostic query syntax without assembling them yourself.

The framework's cross-cutting logic lives in **hooks**: before/after/error middleware attached per service and per method. Authentication, authorization, validation, and data shaping are all hooks. This is Feathers' central tension. The service abstraction is thin and productive, but it pushes security out of the framework and into hooks you must remember to write on every service. The same "any database, common query syntax" convenience means an unfiltered `find` can expose the underlying query engine to external clients. Feathers gives you a lot for free; the discipline of locking it down is entirely yours.

As of 2026 the repository sits around 15.3k stars with a small, stable maintainer group[^2]. It is a mid-size, long-lived project rather than a fast-moving one — active but not hyped, with releases driven by a focused core team. The current line is **Feathers 5 ("Dove")**, a TypeScript-first rewrite that is also the name of the default branch.

## Getting Started

```bash
npm create feathers@latest my-app
cd my-app
npm run dev
```

```ts
// A service is any object with the standard methods.
import { feathers } from '@feathersjs/feathers'

type Message = { id: number; text: string }

class MessageService {
  messages: Message[] = []
  async find() { return this.messages }
  async create(data: Pick<Message, 'text'>) {
    const message = { id: this.messages.length, text: data.text }
    this.messages.push(message)
    return message   // this return value is broadcast to subscribers
  }
}

const app = feathers<{ messages: MessageService }>()
app.use('messages', new MessageService())

// A hook: runs before every 'create' on every service.
app.hooks({
  before: { create: [async (ctx) => { ctx.data.text = ctx.data.text.trim() }] },
})

app.service('messages').on('created', (m) => console.log('new:', m))
await app.service('messages').create({ text: '  hello  ' })
```

The generator (`npm create feathers`) scaffolds a full app with a transport (Koa or Express), Socket.io, an auth service, and a database adapter wired up, which is how most real projects start rather than the bare `feathers()` call above.

## Architecture / How It Works

Four layers compose every Feathers app:

1. **Services** — the unit of business logic. Method names are fixed; Feathers does not care what is behind them (an array, a Mongo collection, a remote API). This uniformity is what lets transports and real-time work generically.
2. **Hooks** — middleware objects `{ before, after, error }` keyed by method. Each hook receives a mutable `context` (`params`, `data`, `result`, `error`). Hooks are where auth, validation, and authorization live. They run in registration order; ordering bugs and forgotten hooks are the usual source of security holes.
3. **Transports** — `@feathersjs/koa` (default in v5) or `@feathersjs/express` expose services over REST; `@feathersjs/socketio` exposes them over websockets and carries real-time events. The same service is reachable both ways with no per-transport code.
4. **Channels** — the real-time publishing layer. Service events are only sent to clients that have been joined to a channel and that the event is published to. If you never configure channels, mutations produce no client events; if you configure them carelessly, users receive records they should not see.

Feathers 5 added **schemas and resolvers** (`@feathersjs/schema`): TypeBox/JSON-Schema definitions for validation plus `resolvers` that populate, redact, or transform data on the way in and out. Resolvers are the intended place to strip secure fields and to enforce per-record access, complementing hooks. **Database adapters** (`@feathersjs/mongodb`, `@feathersjs/knex` for SQL, `@feathersjs/memory`, and community adapters) implement the service methods against a store and share a common query syntax (`$limit`, `$sort`, `$in`, `$or`, and operators like `$gt`). The **client** (`@feathersjs/client`) speaks the identical service API from the browser or React Native, including live-updating results over the socket connection.

## Production Notes

**Query filtering is the primary footgun.** Because adapters accept a rich query syntax and `params.query` often comes straight from an external client, an unguarded `find` lets callers filter, sort, and page over your table freely — and, with some operators, probe data. Feathers ships no default whitelist; you are expected to remove or validate `context.params.query` in a hook (or via schema `queryValidator`) on every externally exposed service. Forgetting this on one service is the classic Feathers vulnerability.

**Authorization is per-service, per-method, and manual.** Authentication (`@feathersjs/authentication`, JWT/local) tells you *who* the caller is; it does not restrict *what* they can touch. Row-level and field-level access are hooks/resolvers you write. There is no framework-wide policy layer, so a new service is wide open until you attach the guards.

**Real-time events are a second authorization surface.** Even if a `find` is locked down, an event published to the wrong channel leaks data. Channel configuration must mirror your access rules, and this is easy to drift out of sync with the hook-based rules on the same service.

**Scaling websockets needs the usual Socket.io machinery.** Multiple instances require sticky sessions at the load balancer and the Socket.io Redis adapter so events fan out across nodes; a single-process default will silently fail to deliver events to clients connected to other instances.

**Adapter behavior is not perfectly uniform.** The common query syntax is a lowest-common-denominator; operator support, `$like`/full-text behavior, default pagination (`$limit`), and transaction semantics differ between the Mongo and SQL/Knex adapters. Portability across databases is real but not total.

**Upgrading 4 → 5 is a rewrite-sized migration.** Dove changed the default transport to Koa, introduced schema/resolvers, moved fully to TypeScript, and reorganized the generator and app structure. Existing Express-based v4 apps keep working (Express is still supported), but adopting the v5 idioms is not a drop-in bump.

## When to Use / When Not

**Use when:**
- You want REST plus real-time sync over CRUD data with minimal glue.
- You want to stay database-agnostic or support several stores behind one API shape.
- You have a live/collaborative UI (chat, dashboards, presence) where automatic mutation events save real work.
- You want the same service API on the server and in a React/React Native/Vue client.

**Avoid when:**
- You need a framework that is secure by default; Feathers trusts you to write the guards.
- Your API does not map cleanly to CRUD-style services (heavy RPC, GraphQL-shaped graphs, complex workflows).
- You want a batteries-included admin UI or content model — that is not what Feathers is.
- You need a large hiring pool or long-term-stable, slow-moving dependency; Feathers is smaller and more niche than the mainstream Node frameworks.

## Alternatives

- nestjs/nest — heavier, DI- and decorator-based; use when you want enforced structure and an enterprise module system rather than thin services.
- strapi/strapi — headless CMS with an admin UI and content modeling; use when the job is managing content, not hand-writing an API.
- supabase/supabase — managed Postgres with built-in realtime, auth, and row-level security; use when you want a hosted backend and DB-enforced authorization instead of hooks.
- expressjs/express or fastify/fastify — bare HTTP frameworks; use when you don't want the service/real-time abstraction and prefer to assemble your own stack.
- directus/directus — data-platform over an existing SQL database; use when you want an instant API plus admin over a schema you already have.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | ~2011–2012 | Open-sourced by David Luecke / Bitovi; services + real-time from the start[^1]. |
| 2.x | 2016 | Pre-3 line; Express-based, Socket.io real-time. |
| 3.0 "Buzzard" | 2017 | New authentication, channels for real-time event publishing[^3]. |
| 4.0 "Crow" | 2020 | Reworked authentication, official TypeScript typings. |
| 5.0 "Dove" | 2023 | TypeScript rewrite, schema/resolvers, Koa default transport, new CLI[^4]. |

## References

[^1]: Feathers "About" / project history. https://feathersjs.com/about/
[^2]: GitHub repository metadata (feathersjs/feathers), fetched 2026-07 — ~15.3k stars, MIT, last push 2026-07-01, default branch `dove`. https://github.com/feathersjs/feathers
[^3]: Feathers 3 "Buzzard" release notes. https://blog.feathersjs.com/
[^4]: Feathers 5 "Dove" migration guide. https://feathersjs.com/guides/migrating.html
[^5]: Feathers guides — services, hooks, channels, schemas. https://feathersjs.com/guides/

## Tags

typescript, javascript, nodejs, api-framework, real-time, rest, websockets, backend, services, hooks, orm-agnostic
