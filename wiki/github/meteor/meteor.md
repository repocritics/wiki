# meteor/meteor

> A full-stack JavaScript platform where the client, server, and database stay in sync over a single reactive data protocol.

[GitHub repo](https://github.com/meteor/meteor) ·
[Official website](https://www.meteor.com) ·
[License: MIT](https://github.com/meteor/meteor/blob/devel/LICENSE)

## Overview

Meteor is a full-stack framework for building web and mobile apps in JavaScript, first released in 2012 by Meteor Development Group (originally under the codename "Skybreak")[^1]. Its defining idea is end-to-end reactivity: a client subscribes to a data set, the server publishes it, and any change to the underlying MongoDB collection is pushed to every subscribed client automatically. The same collection API works on client and server, and client writes apply optimistically before the server confirms them. For its era this collapsed a large amount of glue code — REST endpoints, polling, cache invalidation, form round-trips — into one abstraction.

That integration is also the central tradeoff. Meteor is opinionated in ways most modern JavaScript tooling is not: it ships its own build system, its own wire protocol (DDP), a bundled MongoDB in development, and historically its own package registry. The parts work together tightly, but stepping outside the happy path (a non-Mongo database, a bespoke bundler, horizontal scaling of live queries) means fighting the framework. Meteor peaked in mindshare around 2015–2016 and has since been eclipsed by the React-framework and API-first ecosystems, but it remains actively maintained — the 3.x line is a substantial modernization — and still has a real production install base[^2].

The project is now stewarded by Meteor Software Ltd. after Meteor Development Group spun off Apollo GraphQL as a separate company and the platform was acquired by Tiny in 2019[^3].

## Getting Started

```bash
npx meteor            # install the toolchain on first run
meteor create my-app  # scaffold (prompts for React / Blaze / Vue / Svelte / Solid)
cd my-app
meteor                # dev server on http://localhost:3000, bundled MongoDB
```

A minimal reactive slice — a Method (server RPC) and a Publication (reactive data source):

```js
// server/main.js
import { Meteor } from 'meteor/meteor';
import { Mongo } from 'meteor/mongo';

export const Tasks = new Mongo.Collection('tasks');

Meteor.methods({
  async 'tasks.insert'(text) {
    return await Tasks.insertAsync({ text, createdAt: new Date() });
  },
});

Meteor.publish('tasks', function () {
  return Tasks.find();          // pushes live updates to subscribers over DDP
});
```

```js
// client — subscribe, then read reactively from the local Minimongo cache
Meteor.subscribe('tasks');
const tasks = Tasks.find({}, { sort: { createdAt: -1 } }).fetch();
await Meteor.callAsync('tasks.insert', 'buy milk');  // applies optimistically first
```

## Architecture / How It Works

Meteor is a set of tightly co-designed subsystems rather than a thin library:

- **DDP (Distributed Data Protocol)** — the wire protocol, a JSON-over-WebSocket (SockJS-backed) format for two things: remote method calls (RPC) and publish/subscribe of document sets. It is the spine everything else hangs off.
- **Minimongo** — an in-memory reimplementation of the MongoDB query API that runs in the browser. Subscriptions populate it, and client code queries it synchronously. This is what makes the "same collection API everywhere" claim work.
- **Livequery / oplog tailing** — on the server, publications observe MongoDB for changes. In production this is driven by tailing the Mongo replica-set oplog so writes from anywhere (not just Meteor) propagate to subscribers.
- **Tracker** — a synchronous reactive-dependency system. Any computation that reads reactive data re-runs when that data changes. Blaze and the reactive collection APIs are built on it.
- **Latency compensation** — a Method has a client-side "stub" that runs immediately against Minimongo, giving instant UI feedback; the authoritative server result later reconciles (and rolls back the optimistic write if it diverges).
- **Isobuild** — the build system. It compiles for client, server, and Cordova mobile targets from one source tree, with a plugin model for compilers (e.g. templates, TypeScript).

Historically the most consequential internal was **Fibers**: Meteor used `node-fibers` so server code could call blocking-looking Mongo operations (`Tasks.find().fetch()`) synchronously without callbacks. This shaped the entire API and became a liability as `node-fibers` fell out of step with modern V8. Meteor 3.0 (2024) removed Fibers and moved the server to native `async/await` on top of Express — the largest breaking change in the project's history, requiring the async collection methods (`insertAsync`, `findOneAsync`, etc.) shown above[^4].

The rendering layer is pluggable. **Blaze**, Meteor's own reactive templating engine, was the original default; the project now first-classes React, and supports Vue, Svelte, and Solid through official integrations.

## Production Notes

- **Live-query scaling is the classic footgun.** Every observer of a reactive publication consumes server memory and oplog-processing work. Under many concurrent users with overlapping-but-distinct queries, oplog tailing re-runs a lot of matching on the app server and becomes the bottleneck long before Mongo does. The community `cultofcoders:redis-oplog` package (redirect change notifications through Redis instead of the raw oplog) is the standard mitigation for high-fan-out apps; alternatively, keep hot paths on plain Methods and reserve reactivity for data that genuinely needs it.
- **MongoDB is effectively required.** Reactivity is built on Mongo's oplog; other databases work only as non-reactive data sources with no first-class support. Treat "Meteor" and "Mongo" as a bundle.
- **The 2.x → 3.0 migration is real work.** Removing Fibers means every synchronous server-side Mongo call becomes an `await` of an `*Async` method, and many older Atmosphere packages predate this. Budget for it; do not treat 3.0 as a routine bump. Node 20+ is required (recent 3.x expects Node ≥ 22).
- **Bundle size and build coupling.** Isobuild is the build system; you do not drop in Vite or webpack. It is convenient until you want tooling it does not natively support. Client bundles can grow quietly because packages pull in server+client code paths.
- **Two package worlds.** Meteor predates npm as the default and has its own registry (Atmosphere). Modern Meteor uses npm directly, but you will still meet `meteor add`/Atmosphere packages, and some ecosystem knowledge is split across both.
- **WebSocket-centric deployment.** DDP holds a persistent connection per client. Load balancers must support sticky sessions / WebSockets, and connection count (not just request throughput) is a capacity dimension. Galaxy (the first-party host) is tuned for this; generic PaaS setups need configuration.

## When to Use / When Not

**Use when:**
- You want live, collaborative, data-syncing UIs (dashboards, chat, internal tools) and reactivity is the point.
- You value one integrated toolchain over assembling bundler + API layer + realtime transport yourself.
- Your data model fits MongoDB and your scale is bounded or shardable by subscription.
- You want the same codebase to target web plus iOS/Android via Cordova.

**Avoid when:**
- You need a non-Mongo database as a first-class citizen.
- You expect very high concurrent live-query fan-out and don't want to operate Redis oplog or hand-tune publications.
- You want mainstream, swappable tooling (Vite, your own bundler, edge/serverless runtimes) — Meteor's integration cuts against that.
- You're building a mostly-static or API-first app where a persistent DDP connection buys you nothing.

## Alternatives

- meteor/blaze — extractable reactive templating if you want Meteor's view layer without the platform.
- apollographql/apollo-client — API-first realtime (GraphQL subscriptions) when you want reactivity without Mongo coupling; came out of the same team.
- supabase/supabase — Postgres-based backend with realtime subscriptions and auth; the modern "reactive full-stack" answer on SQL.
- rethinkdb/rethinkdb or feathersjs/feathers — changefeed / service-oriented realtime stacks with looser database coupling.
- vercel/next.js — if you actually want a React framework and realtime is a secondary concern layered on separately.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x ("Skybreak") | 2011–2012 | Prototype; renamed Meteor and open-sourced[^1]. |
| 1.0 | 2014-10 | First stable release. |
| 1.3 | 2016-03 | ES2015 modules and native npm support — major ecosystem shift. |
| 1.5 | 2017-05 | Dynamic imports, bundle-size improvements. |
| 2.0 | 2021-01 | Tree-shaking, Node 12, performance work. |
| 3.0 | 2024 | Fibers removed; server on native async/await + Express; Node 20+ required[^4]. |
| 3.4.x | 2026 | Current maintenance line (Node ≥ 22)[^2]. |

## References

[^1]: Meteor project history and origins. https://www.meteor.com/about
[^2]: Version and Node requirements per the repository README and release badges (Meteor 3.4.1, Node ≥ 22). https://github.com/meteor/meteor
[^3]: Tiny acquisition of Meteor and the Apollo GraphQL spin-off. https://www.meteor.com/blog/tiny-acquires-meteor
[^4]: Meteor 3.0 migration guide — Fibers removal and the move to async/await. https://guide.meteor.com/3.0-migration

## Tags

javascript, full-stack-framework, realtime, reactive, mongodb, ddp, nodejs, meteor, web-framework, rpc, pubsub
