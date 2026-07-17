# sindresorhus/ky

> A tiny HTTP client that wraps the Fetch API with retries, timeouts, JSON handling, and hooks — and nothing else.

[GitHub repo](https://github.com/sindresorhus/ky) ·
[License: MIT](https://github.com/sindresorhus/ky/blob/main/license)

## Overview

Ky is a thin wrapper around the native `fetch` function. It exists because
`fetch` is deliberately low-level: it does not throw on `4xx`/`5xx`, has no
retry or timeout support, requires manual JSON serialization, and offers no way
to set per-instance defaults. Ky adds exactly those ergonomics and stops there —
it is a few kilobytes minified and gzipped, ships as a single package with no
dependencies, and targets modern browsers, Node.js, Bun, and Deno[^1].

The project is authored by Sindre Sorhus, and it carries his house style: a small
surface, opinionated defaults, and an aggressive refusal to grow. Ky will not
become axios. There are no interceptors-as-middleware-stack, no XHR fallback, no
Node-specific transport, no cookie jar. Everything is expressed through `fetch`
options plus a handful of additions (`json`, `retry`, `timeout`, `hooks`,
`searchParams`, a base-URL option). If your problem needs more than that, ky is
the wrong tool by design.

The defining tension is minimalism versus compatibility. Because ky is built on
the platform `fetch`, it inherits every limitation of `fetch` — including that it
was ESM-only and browser-first for years. Ky itself is **ESM-only**, which is the
single most common source of friction for teams still on CommonJS[^2].

## Getting Started

```sh
npm install ky
```

```js
import ky from 'ky';

// POST JSON and parse the response in one chain — no manual await on Response,
// no JSON.stringify, and a thrown HTTPError on any non-2xx status.
const data = await ky.post('https://example.com/api/users', {
  json: {name: 'Ada'},
}).json();
```

Instances with shared defaults are the idiomatic pattern:

```js
import ky from 'ky';

const api = ky.extend({
  timeout: 5000,
  retry: {limit: 2},
  hooks: {
    beforeRequest: [
      request => {
        request.headers.set('Authorization', `Bearer ${token}`);
      },
    ],
  },
});

const user = await api.get('users/1').json();
```

## Architecture / How It Works

Ky is a factory around `fetch`. `ky(input, options)` returns a `Promise` that is
also decorated with body-parsing shortcuts — `.json()`, `.text()`, `.blob()`,
`.arrayBuffer()`, `.formData()`, and `.bytes()` (where the runtime supports it) —
so you can call `ky.get(url).json()` without first awaiting the `Response`. When
you use one of these shortcuts, ky sets an appropriate `Accept` header. Unlike
raw `fetch`, these throw an `HTTPError` when the status is outside `200–299`, and
`.json()` also throws on an empty body or `204`.

Key internals:

- **Error model.** Non-2xx responses (after redirects are followed) become an
  `HTTPError` carrying the `Request` and `Response`. Network failures surface as
  the underlying error; timeouts throw a `TimeoutError`.
- **Retries.** Idempotent methods (`get`, `put`, `head`, `delete`, `options`,
  `trace`, and `query`) retry by default up to a limit of 2, on a fixed set of
  status codes (`408`, `413`, `429`, `500`, `502`, `503`, `504`) with exponential
  backoff. `Retry-After` and rate-limit headers are honored for the applicable
  codes. Network errors are retried for retriable methods.
- **Hooks.** Five lifecycle arrays — `init`, `beforeRequest`, `beforeRetry`,
  `beforeError`, and `afterResponse` — run serially. `beforeRequest` can return a
  `Request` to replace the outgoing one, or a `Response` to short-circuit the
  network entirely (useful for mocking or a cache layer).
- **Timeout.** Implemented with `AbortController` under the hood; the default is
  a 10s per-attempt timeout.
- **Instances.** `ky.extend()` merges new defaults onto an existing instance;
  `ky.create()` starts fresh. Headers and hooks merge deeply, which is what makes
  layered API clients composable.

There is no transport abstraction. Ky is `fetch` all the way down, which is why it
is small and why its behavior is exactly the platform's behavior plus a documented
delta.

## Production Notes

- **ESM-only.** `require('ky')` does not work. On CommonJS you must use dynamic
  `import()`, or pin the last CommonJS-capable release (`ky@0.33`). This trips up
  Jest/older-toolchain setups more than anything else about the library[^2].
- **Retries buffer the request body in memory.** When retries are enabled, ky
  clones the body before each attempt via `ReadableStream.tee()`, which buffers
  the whole stream. For large streaming uploads this is a memory footgun — set
  `retry: {limit: 0}` when uploading big bodies you don't need to retry[^3].
- **Timeouts are per-attempt, not per-operation** by default. Three retries with a
  10s timeout can run far longer than 10s wall-clock. If you need a hard ceiling
  across retries and backoff delays, use the overall-timeout option rather than
  assuming `timeout` bounds the whole call.
- **`fetch` must exist.** Node gained a stable global `fetch` in Node 18, so
  modern Node needs no polyfill; older Node required the companion `ky-universal`
  package, which is now deprecated. In non-`fetch` environments you must provide a
  `fetch` implementation.
- **Browser double-retry.** Chromium retries `408` on keep-alive connections at
  the network layer, so a `408` can be retried by both the browser and ky. Drop
  `408` from the retry status codes, or set `keepalive: false`, if duplicate
  attempts matter[^3].
- **No cancellation of in-flight retries beyond the standard `signal`.** You pass
  an `AbortSignal` like you would to `fetch`; ky wires it through the retry loop.

## When to Use / When Not

**Use when:**
- You already target `fetch` (browser, Bun, Deno, or Node 18+) and want retries,
  timeouts, JSON, and shared defaults without hand-rolling them.
- Bundle size matters — ky is a few KB and dependency-free.
- You want a client whose behavior is "`fetch`, but ergonomic," with no surprises.

**Avoid when:**
- You are on CommonJS and cannot use dynamic `import()`.
- You need Node-specific power features: streams, pagination, cookie jars, HTTP/2
  tuning — reach for got instead.
- You need broad legacy-environment support or an XHR fallback (upload progress in
  environments without `fetch`), which axios provides.
- You want a large, batteries-included client with request/response interceptor
  ecosystems.

## Alternatives

- axios/axios — feature-rich, interceptors, wide environment support, CommonJS-friendly; use when compatibility and features beat size.
- sindresorhus/got — same author, Node-only, far more powerful (streams, pagination, retries, hooks); use for serious server-side HTTP.
- unjs/ofetch — isomorphic `fetch` wrapper used by Nuxt; use when you want auto-JSON and a Nuxt-adjacent stack.
- elbywan/wretch — fluent, chainable `fetch` wrapper in a similar tiny niche; use when you prefer a builder API.
- developit/redaxios — the axios API reimplemented on `fetch` in ~1 KB; use as a near drop-in for axios with a smaller footprint.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-09 | First release: `fetch` wrapper with retries and JSON[^1]. |
| ESM-only | ~2022 | Package converted to ESM-only; `ky@0.33` is the last CommonJS-capable line[^2]. |
| 1.0 | 2023 | First stable major; consolidated the ESM-only, `fetch`-based API. |
| next | in progress | README documents an upcoming line adding a base-URL option, Standard Schema response validation, `totalTimeout`, retry jitter, and a `QUERY` shortcut[^4]. |

## References

[^1]: ky README and repository metadata — description "Tiny & elegant JavaScript HTTP client based on the Fetch API," created 2018-09-04. https://github.com/sindresorhus/ky
[^2]: ky is published as ESM-only; `require()` is unsupported and dynamic `import()` or an older release is needed on CommonJS. https://github.com/sindresorhus/ky#install
[^3]: ky README, retry notes — body buffering via `tee()` and the Chromium `408` double-retry caveat. https://github.com/sindresorhus/ky#retry
[^4]: ky README install note — "This readme is for the next version of Ky," documenting base-URL, Standard Schema validation, `totalTimeout`, and jitter options. https://github.com/sindresorhus/ky

## Tags

javascript, typescript, http-client, fetch, rest, browser, nodejs, deno, bun, esm, tiny
