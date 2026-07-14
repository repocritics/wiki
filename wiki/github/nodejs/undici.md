# nodejs/undici

> An HTTP/1.1 client written from scratch for Node.js — and the engine behind Node's built-in `fetch`.

[GitHub repo](https://github.com/nodejs/undici) ·
[Official website](https://undici.nodejs.org/) ·
[License: MIT](https://github.com/nodejs/undici/blob/main/LICENSE)

## Overview

Undici is a low-level HTTP/1.1 client for Node.js, started in 2018 by Matteo Collina and now maintained under the `nodejs` org[^1]. The name is Italian for "eleven" — a pun on HTTP/1.1 (and a Stranger Things reference). Unlike `got`, `axios`, or `node-fetch`, it does not wrap Node's built-in `http` module; it speaks the wire protocol directly over sockets, which is where most of its throughput advantage comes from.

Its reach is much larger than its star count (~7.6k) suggests. Since Node.js 18 (2022), the global `fetch`, `Headers`, `Response`, `Request`, `FormData`, `WebSocket`, and `EventSource` are all implemented by a bundled copy of undici[^2]. So virtually every modern Node process already runs undici whether the author installed it or not. The standalone npm package exists to expose the faster low-level API (`request`, `stream`, `pipeline`, `dispatch`) and features that the WHATWG `fetch` surface deliberately hides — connection-pool tuning, interceptors, `ProxyAgent`, `MockAgent`.

The defining tension is spec-compliance versus performance. `undici.fetch()` is a faithful, and therefore relatively heavy, implementation of the WHATWG Fetch standard (Web Streams, body cloning, spec-mandated buffering). `undici.request()` is the same HTTP core with a Node-native, non-spec API that is several times faster. Choosing undici well means knowing which of its two front doors you are walking through.

## Getting Started

```bash
npm i undici
```

```js
import { request } from 'undici'

// Low-level API — fastest path, Node-native ergonomics
const { statusCode, headers, body } = await request('https://api.example.com/data')
console.log(statusCode)
const json = await body.json()   // body is a consumable stream + mixins
```

```js
import { Agent, setGlobalDispatcher } from 'undici'

// Tune the connection pool that all requests (and global fetch) will use
setGlobalDispatcher(new Agent({
  keepAliveTimeout: 10_000,
  connections: 128,       // max sockets per origin
  pipelining: 1,          // requests in flight per socket
}))
```

## Architecture / How It Works

The central abstraction is the **Dispatcher**. Everything — `fetch`, `request`, `stream`, `pipeline` — ultimately calls `dispatcher.dispatch()`. Concrete dispatchers form a hierarchy:

- **Client** — a single persistent connection to one origin. The lowest unit; holds one socket, an outgoing request queue, and an HTTP/1.1 parser.
- **Pool** — a set of `Client`s to the same origin, load-balanced across sockets. This is what most apps actually want for one host.
- **BalancedPool** — spreads load across multiple upstream origins.
- **Agent** — the default global dispatcher; lazily creates a `Pool` per origin as new hosts are requested. `getGlobalDispatcher()` / `setGlobalDispatcher()` swap it.

The HTTP/1.1 parser is a WebAssembly build of `llhttp` (the same parser Node core uses), which avoids per-byte JS overhead. Sockets are kept alive and, optionally, **pipelined** (multiple requests sent before responses return) — off by default because many real-world servers and proxies mishandle it.

**Interceptors** wrap a dispatcher via `.compose()`, forming a middleware chain: redirect following, retry, response caching, DNS caching, and dump-on-abort all ship as composable interceptors rather than being baked into the core. This keeps the hot path lean and makes behavior opt-in.

`undici.fetch()` is built *on top of* this dispatcher core as a separate spec-compliance layer: it adds Web Streams bodies, CORS-mode semantics (mostly inert in Node), and the `Request`/`Response`/`Headers` classes. HTTP/2 exists but is opt-in per-client via the `allowH2` option and is less mature than the HTTP/1.1 path[^3].

## Production Notes

- **`fetch` vs `request` is a real performance decision.** The Fetch spec forces body buffering and Web Streams; `undici.request()` skips all of it. On the project's own benchmark, `undici.request`/`stream`/`dispatch` run 2–4× the throughput of `undici.fetch`[^4]. If you control both ends and don't need spec semantics, use `request`.
- **A response body is a stream you must consume.** Ignoring `body` leaks the socket back-pressure and can stall the pool. If you don't need the body, call `body.dump()` or consume it. This is the single most common undici footgun.
- **Global-vs-installed version skew.** `process.versions.undici` is the copy baked into your Node runtime; `npm i undici` layers a possibly-newer copy on top. Mixing them causes subtle bugs — most notoriously passing a global `FormData` to `undici.fetch()` (or vice-versa), which fails because the two `FormData` classes differ. Use `install()` to force all globals to the installed copy, or keep both from one source[^2].
- **Timeouts are opt-in and multi-layered.** `headersTimeout` and `bodyTimeout` default to non-infinite but generous values; `connect.timeout` is separate. A "hung request" is usually a body that never finished streaming, not a dead socket. Set these explicitly for outbound calls to third parties.
- **Testing.** `MockAgent` intercepts at the dispatcher layer, so it works transparently for both `fetch` and `request` once set as the global dispatcher — the standard way to stub HTTP in Node tests without monkey-patching.
- **Observability.** Undici publishes to `diagnostics_channel` (request/response/error/connect events), which is how APM tools instrument it. There is no built-in logging.
- **Proxies need `ProxyAgent`.** Undici does *not* read `HTTP_PROXY`/`HTTPS_PROXY` environment variables automatically; you must construct and install a `ProxyAgent` yourself.

## When to Use / When Not

**Use when:**
- You want the fastest HTTP client in Node and can use the `request`/`stream`/`dispatch` API.
- You need connection-pool control, pipelining, per-origin tuning, or custom interceptors.
- You need `ProxyAgent`, `MockAgent`, or a standards-compliant `WebSocket`/`EventSource` in Node.
- You want to pin a newer `fetch`/undici than your Node runtime bundles.

**Avoid when:**
- You just need occasional requests — the built-in global `fetch` is already undici and needs no dependency.
- You want browser portability — write to the Web `fetch` API, not undici's Node-native methods.
- You need a batteries-included client with automatic retries, hooks, and pagination out of the box — `got` and `axios` are higher-level.
- You need mature first-class HTTP/2 or HTTP/3; undici is HTTP/1.1-first.

## Alternatives

- sindresorhus/got — higher-level Node client with retries, hooks, pagination; built on Node http, slower but more ergonomic for app code.
- axios/axios — isomorphic (browser + Node) client with interceptors and a huge install base; use when you want one API across environments.
- node-fetch/node-fetch — the pre-Node-18 `fetch` polyfill; now largely obsoleted by built-in `fetch` (which is undici).
- nodejs/node built-in `fetch` / `http` — use when you want zero dependencies and standard-only APIs.
- szmarczak/http2-wrapper — reach for when HTTP/2 is the primary requirement rather than an afterthought.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo start | 2018-05 | First commit; from-scratch HTTP/1.1 client experiment[^1]. |
| bundled in Node 18 | 2022-04 | Global `fetch` ships (experimental) powered by undici[^2]. |
| stable `fetch` in Node 21 | 2023-10 | Global `fetch` unflagged as stable in Node core. |
| 6.0 | 2024 | Major: dropped older Node versions, API cleanup, tree-shakeable build. |
| 7.0 | 2024–2025 | Major: interceptor/dispatcher API refinements, further removals. |

## References

[^1]: undici repository and history. https://github.com/nodejs/undici
[^2]: undici README — "Undici vs. Fetch" and `install()` globals. https://github.com/nodejs/undici/blob/main/README.md
[^3]: undici docs — Dispatcher / Client (`allowH2`, connection options). https://undici.nodejs.org/
[^4]: undici README benchmarks (Node 24, 50 connections, pipelining 10). https://github.com/nodejs/undici/blob/main/README.md#benchmarks

## Tags

javascript, nodejs, http-client, http, fetch, networking, connection-pool, web-standards, performance, library
