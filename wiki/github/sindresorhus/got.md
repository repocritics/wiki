# sindresorhus/got

> A Node.js HTTP client built on the core `http`/`https` modules, with retries, streams, pagination, and RFC 7234 caching built in.

[GitHub repo](https://github.com/sindresorhus/got) ·
[License: MIT](https://github.com/sindresorhus/got/blob/main/license)

## Overview

Got is an HTTP request library for Node.js, first published in 2014 as a small alternative to the then-dominant `request` library[^1]. Over a decade it grew from a thin wrapper around Node's `http.request` into a feature-heavy client with automatic retries, a unified stream/promise API, cursor-and-link pagination, HTTP/2, hook-based extensibility, and standards-compliant response caching. It is written in TypeScript (rewritten from JavaScript in the v10 line, 2019) and maintained by Sindre Sorhus and Szymon Marczak.

The defining fact about Got in 2026 is that its own README steers new users elsewhere: "You probably want Ky instead"[^2]. Ky, by the same authors, is smaller, runs in the browser, and is built on the Fetch API. Got is now positioned for cases that genuinely need its Node-specific machinery — streaming, low-level timeout phase control, request timings, RFC 7234 caching — rather than as the default HTTP client. That candor is unusual and worth taking at face value: reach for Got when you need what it uniquely provides, not by habit.

The other defining fact is packaging. Got is native [ESM](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) only since v12 and ships no CommonJS build[^3]. A CommonJS codebase cannot `require('got')`; you either convert to ESM, use dynamic `import()`, or stay on the unmaintained v11 line. This single decision is the most common reason projects pin Got or migrate away.

## Getting Started

```sh
npm install got
```

```js
import got from 'got';

// JSON POST with typed response and automatic parsing
const data = await got.post('https://httpbin.org/anything', {
	json: {hello: 'world'},
}).json();

console.log(data.json); //=> {hello: 'world'}
```

```js
// Streaming a download to disk — Got exposes a Node stream directly
import {pipeline} from 'node:stream/promises';
import fs from 'node:fs';
import got from 'got';

await pipeline(
	got.stream('https://example.com/large.bin'),
	fs.createWriteStream('large.bin'),
);
```

## Architecture / How It Works

Got is built on Node's core `http`/`https`/`http2` modules, not on `fetch` or `undici`. This is the root of both its strengths (deep access to connection timings, per-phase timeouts, agents, Unix sockets) and its limitations (Node-only, no browser target).

The central design choice is that a Got call returns a hybrid object. `got(url)` yields a promise that also behaves as a duplex stream, and `.json()` / `.text()` / `.buffer()` on that promise choose a body-parsing mode. `got.stream(url)` requests the stream side explicitly. Internally the request is a state machine that can be retried, redirected, and resumed; the promise and stream surfaces are two views over the same lifecycle.

Notable subsystems:

- **Retry** is on by default. Idempotent methods and a fixed set of network/HTTP error codes trigger exponential backoff with jitter, capped by `retry.limit` (default 2). This is a frequent surprise — a request that appears to hang for seconds is often Got silently retrying. Set `retry.limit: 0` to disable[^4].
- **Timeouts** are phase-based, not a single deadline. You can bound `lookup`, `connect`, `secureConnect`, `socket`, `response`, `send`, and total `request` independently[^5]. This granularity is one of Got's genuine differentiators over `fetch`.
- **Hooks** (`beforeRequest`, `beforeRetry`, `beforeRedirect`, `afterResponse`, `beforeError`) allow mutation of the request/response and are how instances add auth, logging, or signing.
- **Instances** via `got.extend(options)` produce a new client with merged defaults; `got.extend(a, b)` composes them. This is the intended extensibility model — the plugin ecosystem (`gh-got`, `got-scraping`, `got4aws`) is built on it.
- **Caching** implements RFC 7234 via a pluggable `cacheable-request` + `keyv` store, so responses can be cached in memory, Redis, or SQLite with correct `Cache-Control`/`ETag` handling — rare in the JS HTTP-client space.

## Production Notes

**ESM-only is the dominant migration cost.** Since v12, there is no CommonJS entry point. In a CJS project your options are: `const {default: got} = await import('got')` inside an async context, migrate the project to ESM, or pin `got@11`. Jest and ts-node setups frequently break on this because they transpile `import` to `require` unless configured for native ESM. Budget real time for it.

**Retries are silent and can amplify load.** Because retry is on by default with backoff, a downstream outage can turn one logical request into several, and a slow endpoint can look like a client-side stall. In latency-sensitive or high-fan-out services, set an explicit `retry.limit` and phase `timeout` rather than accepting defaults.

**Install size and dependency count are non-trivial.** Got pulls in a tree (`@sindresorhus/is`, `cacheable-request`, `keyv`, `decompress-response`, `responselike`, and others). For a service that only needs a couple of GET/POST calls, this is heavier than native `fetch` (built into Node 18+) or Ky. The maintainers' own recommendation of Ky / fetch-extras for "simple needs" reflects this.

**Version support is short.** v11 is explicitly unmaintained and backport requests are refused[^3]. Staying on v11 for CommonJS reasons means running a client that will not receive security fixes — a real consideration for a component that makes outbound network requests.

**No browser build.** Got is Node-only by design. Anything targeting browsers, edge runtimes, Deno, or Bun's web-standard surface should use Ky or native fetch. `got-fetch` exists to expose a fetch-like interface but does not change the Node-only footprint.

## When to Use / When Not

**Use when:**
- You need streaming uploads/downloads with progress events on Node.
- You need per-phase timeout control or connection timings (`response.timings`).
- You need RFC 7234-compliant response caching out of the box.
- You need built-in cursor/link pagination or retry-with-backoff without hand-rolling it.

**Avoid when:**
- Your project is CommonJS and you cannot adopt ESM — the friction is not worth it.
- You target the browser, Deno, Bun, or edge runtimes — use Ky or fetch.
- You only make a handful of simple requests — native `fetch` (Node 18+) has no dependency cost.
- You want the fastest possible throughput — `undici` is generally faster for high-volume Node HTTP.

## Alternatives

- sindresorhus/ky — use instead when you want a smaller, Fetch-based client that also runs in the browser (the authors' own recommendation for most cases).
- nodejs/undici — use instead when you need maximum Node HTTP throughput; it is the engine behind Node's built-in `fetch`.
- axios/axios — use instead when you want one isomorphic API across browser and Node with the widest ecosystem familiarity.
- node-fetch/node-fetch — use instead when you want a minimal Fetch API in Node, though native `fetch` has largely superseded it.
- Native `fetch` (Node 18+) — use instead when a couple of requests don't justify any dependency at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015 | Small wrapper over `http`/`https`, alternative to `request`[^1]. |
| 9.x | 2018 | Hooks, instances, retry maturation. |
| 10.0 | 2019-11 | Full rewrite in TypeScript; API overhaul[^6]. |
| 11.0 | 2020 | HTTP/2 support, pagination API, advanced HTTPS options. Last CommonJS line. |
| 12.0 | 2022 | Native ESM only; CommonJS export dropped[^3]. |
| 13.0 | 2023 | Dependency and option cleanups on the ESM base. |
| 14.0 | 2024 | Continued ESM-line maintenance and modernization. |

(Major-version dates below the 10.0 rewrite are given at year granularity where a precise release date is not verified.)

## References

[^1]: `got` on npm — first published 2014; package history. https://www.npmjs.com/package/got
[^2]: Got README, top-of-file recommendation of Ky. https://github.com/sindresorhus/got#readme
[^3]: Got README, "Warning: This package is native ESM" and "Got v11 is no longer maintained." https://github.com/sindresorhus/got#install
[^4]: Got retry documentation. https://github.com/sindresorhus/got/blob/main/documentation/7-retry.md
[^5]: Got timeout documentation. https://github.com/sindresorhus/got/blob/main/documentation/6-timeout.md
[^6]: Got 10 release — TypeScript rewrite. https://github.com/sindresorhus/got/releases

## Tags

typescript, javascript, nodejs, http-client, http-request, https, esm, networking, streaming, retry, http2
