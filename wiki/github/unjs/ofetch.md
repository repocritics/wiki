# unjs/ofetch

> A thin ergonomic layer over native `fetch` — auto JSON, thrown errors,
> retries, and interceptors — and the engine behind Nuxt's `$fetch`.

[GitHub repo](https://github.com/unjs/ofetch) ·
[License: MIT](https://github.com/unjs/ofetch/blob/main/LICENSE)

## Overview

ofetch is the HTTP client of the UnJS ecosystem (Pooya Parsa and the Nuxt
core team), started in December 2020 as `ohmyfetch` and renamed to `ofetch`
at v1.0.0 (2022-11-15)[^1]. It does not replace `fetch` — it wraps whatever
`fetch` the runtime provides (browser, Node ≥18 via undici, Deno, Bun,
Cloudflare Workers) and fixes the ergonomics the WHATWG API refuses to:
responses are JSON-parsed automatically, non-2xx statuses throw a
`FetchError` with the parsed body attached, request bodies are stringified,
and retry, timeout, `baseURL`, and query merging are options rather than
hand-rolled boilerplate[^2].

Its 5.3k stars understate its reach: ofetch is the implementation of `$fetch`
in Nuxt 3 and the fetch utility inside Nitro, so every Nuxt app ships it, and
npm downloads run far ahead of its GitHub visibility. Maintenance is active
(pushed within days as of mid-2026). The defining tradeoff is convenience
versus transparency: ofetch consumes the response body and returns parsed
data, not a `Response`. That is what you want ~90% of the time; the rest
(headers, status, manual streaming) requires knowing the escape hatches
(`ofetch.raw`, `ofetch.native`, `responseType: "stream"`).

## Getting Started

```bash
npm install ofetch
```

```ts
import { ofetch, FetchError } from "ofetch";

// GET, JSON parsed, typed (assertion only — no runtime validation)
const todo = await ofetch<{ id: number; title: string }>(
  "https://jsonplaceholder.typicode.com/todos/1",
);

// Instance with shared defaults and interceptors;
// object bodies are auto-stringified with content-type set for you
const api = ofetch.create({
  baseURL: "https://api.example.com",
  retry: 2,
  retryDelay: 500,
  async onRequest({ options }) {
    options.headers.set("Authorization", `Bearer ${getToken()}`);
  },
});
await api("/users", { query: { page: 2 } })
  .catch((err: FetchError) => console.error(err.status, err.data));
```

## Architecture / How It Works

v1 is a small TypeScript codebase over three UnJS dependencies: **ufo** for
URL work (`baseURL` joining, `query` merging with slash/encoding edge cases),
**destr** for JSON parsing (prototype-pollution-safe, falls back to raw text
instead of throwing on invalid JSON), and **node-fetch-native** as a
conditional `fetch` polyfill for older Node[^2]. v2 deletes all three:
zero dependencies, ESM-only, native `JSON.parse` and inlined URL utilities,
shrinking install size from ~900 KB to ~28 KB[^3].

The request pipeline: normalize options (since v1.4, `headers` is always a
`Headers` instance inside interceptors[^4]) → `onRequest` interceptor(s) →
native `fetch` → sniff `content-type` to pick a parser (JSON via destr,
binary as `Blob`, `text/event-stream` as a stream since v1.5[^5]; override
via `responseType` or `parseResponse`) → if `response.ok` is false, run
`onResponseError` and throw `FetchError`; else `onResponse`, return body.

Retry triggers only on a status-code allowlist (408, 409, 425, 429, 500,
502–504 by default; configurable via `retryStatusCodes`): one retry for
idempotent methods, **zero** for POST/PUT/PATCH/DELETE — but any explicit
`retry` value re-enables retries for all methods[^2]. `ofetch.create`
inherits defaults with a one-level clone; interceptors can be arrays and
compose across `create` layers (since v1.4[^4]). `ofetch.raw` returns the
`Response` with the parsed body on `response._data`; `ofetch.native` is the
untouched runtime `fetch`.

## Production Notes

- **`ofetch<T>()` is a cast, not validation.** The generic asserts the shape;
  a server returning garbage still "succeeds" typed. Pair with zod/valibot at
  trust boundaries.
- **Version/docs skew.** `main`'s README describes v2 alpha; on npm `latest`
  (v1.5.x) read the `v1` branch docs. v2 is ESM-only and drops the Node <18
  polyfill — CJS consumers cannot upgrade without a build change[^3].
- **Custom `retry` retries mutations.** Setting `retry: 3` explicitly applies
  to POST too; scope retries per call, not in a shared `create` instance. No
  exponential backoff built in — `retryDelay` is a fixed number (a callback
  form landed in v1.4[^4]).
- **Body is consumed.** By the time an error throws, the body has been read
  into `error.data`; you cannot re-stream it. For large downloads use
  `responseType: "stream"` or `ofetch.raw` to avoid buffering everything.
- **Shallow defaults merge.** `create` clones one level; nested option
  objects (notably `headers` pre-1.4) can be shared or clobbered across
  instances. And at 1.4, `ctx.options.headers` became a normalized `Headers`
  object — interceptors assuming a plain object broke despite the release
  being labeled non-breaking[^4].
- **Node proxies are not automatic.** `HTTP_PROXY` env vars are ignored by
  Node's fetch unless `NODE_USE_ENV_PROXY=1`; otherwise pass an undici
  `ProxyAgent` via the `dispatcher` option[^2].
- **No caching or deduplication.** Nuxt's `useFetch` adds SSR payload reuse
  and dedup on top; ofetch itself fires a network request every call.

## When to Use / When Not

**Use when:**
- You are in the Nuxt/Nitro/UnJS stack — it is already your `$fetch`.
- You need one HTTP client across browser, Node, workers, and edge runtimes.
- You want `fetch` semantics plus JSON/error/retry ergonomics at single-digit
  KB, instead of axios's XHR-era weight.

**Avoid when:**
- You need deep Node HTTP control (pooling, HTTP/2, streams-first APIs) —
  use undici directly or got.
- You need built-in caching, request dedup, or upload progress events.
- You need runtime response validation — add a schema layer or use an
  RPC-style client (tRPC, oRPC).

## Alternatives

- axios/axios — use instead for upload/download progress events or its large
  middleware/adapter ecosystem; heavier and not fetch-native.
- sindresorhus/ky — closest peer: tiny fetch wrapper with hooks and retries;
  pick it outside the UnJS ecosystem, browser/Deno-first.
- sindresorhus/got — Node-only, deeper retry/pagination/agent control for
  server-to-server workloads.
- elbywan/wretch — fluent chainable API over fetch, if you prefer builder
  style to an options object.
- nodejs/undici — the engine under Node's fetch; use directly for maximum
  throughput and pool tuning, at the cost of all ergonomics.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x (`ohmyfetch`) | 2020-12 | Created as `ohmyfetch`[^1]. |
| 1.0.0 | 2022-11-15 | Renamed to `ofetch`[^1]. |
| 1.1.0 | 2023-06-06 | `ignoreResponseError`, deep option merge, runtime export conditions[^6]. |
| 1.3.0 | 2023-08-23 | Configurable `retryStatusCodes`, auto `duplex` for streamed bodies[^7]. |
| 1.4.0 | 2024-09-20 | Interceptor arrays, `retryDelay` callback, headers normalized to `Headers`[^4]. |
| 1.5.0 | 2025-10-28 | SSE auto-detected as stream, `params` deprecated for `query`[^5]. |
| 2.0.0-alpha | 2025-10-28 | On `main`: zero-dependency, ESM-only, custom `AbortSignal` + timeout[^3]. |

## References

[^1]: ofetch v1.0.0 tag (rename from ohmyfetch) — 2022-11-15. https://github.com/unjs/ofetch/releases
[^2]: ofetch README (v1/v2 branches). https://github.com/unjs/ofetch#readme
[^3]: ofetch v2.0.0-alpha.1 release notes — 2025-10-28. https://github.com/unjs/ofetch/releases/tag/v2.0.0-alpha.1
[^4]: ofetch v1.4.0 release notes — 2024-09-20. https://github.com/unjs/ofetch/releases/tag/v1.4.0
[^5]: ofetch v1.5.0 release notes — 2025-10-28. https://github.com/unjs/ofetch/releases/tag/v1.5.0
[^6]: ofetch v1.1.0 release notes — 2023-06-06. https://github.com/unjs/ofetch/releases/tag/v1.1.0
[^7]: ofetch v1.3.0 release notes — 2023-08-23. https://github.com/unjs/ofetch/releases/tag/v1.3.0

## Tags

typescript, http-client, fetch, nodejs, browser, edge-runtime, unjs, nuxt, isomorphic, networking
