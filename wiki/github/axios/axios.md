# axios/axios

The promise-based HTTP client for browser and Node.js — for years the default `npm install` for JS HTTP requests, now competing with `fetch` and modern alternatives.

## What it is

A JavaScript library that wraps XMLHttpRequest (in browsers) and `http`/`https` (in Node.js) into a consistent promise-based API. Predates `fetch`-as-default-in-Node by years, which is why it became the canonical HTTP client across the npm ecosystem. Provides request/response interceptors, automatic JSON transforms, request cancellation, and broad config options out of the box.

## Key features

- Same API in browser and Node — request/response interceptors, auto-JSON, timeouts, retries (via plugins).
- Request/response transformation pipeline (interceptors).
- Cancellation via AbortController + the older CancelToken pattern.
- Form-data / multipart handling baked in.
- Custom error class with structured response metadata (status, headers, data).
- Plugin ecosystem: axios-retry, axios-cache-interceptor, etc.
- MIT-licensed.

## Tech stack

- JavaScript primary; TypeScript types bundled.
- npm packages: `axios` (main).
- Default branch is `v1.x` — released versions live on a stable major branch.

## When to reach for it

- You need a single HTTP client API across browser + Node code (legacy Node, before fetch was stable).
- You want interceptors for cross-cutting concerns (auth headers, error handling, logging).
- You're integrating with a JS codebase where Axios is already the convention.

## When *not* to reach for it

- You're targeting modern environments — `fetch` is now stable in Node 18+ and all browsers.
- Bundle size matters — axios adds ~13KB gzipped vs. 0 for `fetch`.
- You want minimal API surface — `ky`, `wretch`, or `ofetch` are lighter modern alternatives.

## Maturity signal

109k stars, 12k forks, MIT, last push 2026-06-01. 12-year-old project; maintenance has stabilized after some governance churn in the early 2020s. Open-issues count of 146 is low for a library of this scope, signaling tight triage. Continues to be the safe default for codebases that don't want to fight ecosystem momentum.

## Alternatives

- Native `fetch` — use for new code targeting modern environments.
- `ky`, `wretch`, `ofetch` — use when you want a smaller, fetch-based wrapper.
- `got` (Node-only) — use when you need advanced retry / stream features in Node.
- `undici` — use when you want the fastest Node HTTP client.

## Notes

The "should I still use axios?" question is the recurring debate. For new code in 2026, native fetch + a thin wrapper covers most use cases; axios's value remains in legacy codebases and apps that lean heavily on interceptors. The `v1.x` default branch signals the team's preference for stable-release tracking rather than continuous main-branch churn.

## Tags

javascript, typescript, http-client, library, nodejs, browser, promise, awesome-list
