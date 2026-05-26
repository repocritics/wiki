# nodejs/node

> Node.js — the V8 JavaScript runtime that defined server-side JavaScript.

[GitHub repo](https://github.com/nodejs/node) ·
[Official website](https://nodejs.org) ·
[License: MIT](https://github.com/nodejs/node/blob/main/LICENSE)

## Overview

Node.js is the OpenJS Foundation–governed JavaScript runtime built on V8 (Chrome's JS engine) and libuv (the cross-platform async I/O library)[^1]. Since Ryan Dahl's 2009 introduction, Node has been the dominant server-side JavaScript runtime; the LTS release line (currently Node 22 LTS, with Node 24 entering LTS in late 2025) defines the de facto baseline for npm packages and tooling.

Modern Node has steadily absorbed features long associated with competitors (Bun, Deno): a built-in test runner (`node:test`), `node --watch`, `node --run`, native ESM, a global `fetch`, the Web Streams API, single-executable applications (`node:sea`), and an experimental permission model (`--permission`). Most of the post-2022 work has been about closing the perceived ergonomics gap, not pure performance.

The most significant ongoing project in 2025 is Node's incremental migration toward V8 sandbox + permission-by-default models, alongside ongoing CommonJS deprecation messaging. CommonJS is not going away in any near-term Node release, but every new feature is ESM-first.

## Getting Started

Install via `nvm`, `fnm`, or `volta`:

```bash
# fnm (fastest)
curl -fsSL https://fnm.vercel.app/install | bash
fnm install 22
fnm use 22
```

A minimal HTTP server using the modern global `fetch`-compatible APIs:

```js
// server.mjs
import { createServer } from "node:http"

const server = createServer((req, res) => {
  res.end("Hello from Node")
})

server.listen(3000)
```

```bash
node --watch server.mjs
```

Built-in test runner:

```js
// math.test.mjs
import { test } from "node:test"
import assert from "node:assert/strict"

test("adds two numbers", () => {
  assert.equal(1 + 2, 3)
})
```

```bash
node --test
```

Run an npm script without a script runner:

```bash
node --run dev
```

## Architecture / How It Works

Node's runtime has three layers[^2]:

1. **V8** — the JavaScript engine. Node embeds V8 directly and exposes its C++ API for native addons (via N-API for ABI stability).
2. **libuv** — the cross-platform async I/O library. Provides the event loop, thread pool (default 4 workers), and async file / network / process primitives.
3. **Node core** — modules implemented in JavaScript or C++ bindings (`fs`, `http`, `crypto`, `stream`, etc.). The `node:` prefix is now the recommended import form.

The **event loop** has well-defined phases: timers → pending callbacks → idle/prepare → poll → check → close. Microtasks (Promises) run between phases. Understanding this is essential for debugging "why didn't my timer fire" or "why is my Promise resolving in the wrong order" issues.

**ESM and CommonJS** coexist with rough edges. A `.mjs` file or `"type": "module"` package gets ESM semantics; `.cjs` or default-typed packages get CommonJS. Cross-importing CJS from ESM works via default-import shimming; ESM from CJS works via dynamic `import()`. Node 22 added `require(esm)` for synchronous import of fully-static ESM, finally smoothing the largest remaining wart.

**Worker threads** (`node:worker_threads`) provide true parallelism via separate V8 isolates with `SharedArrayBuffer` and `MessagePort`-based communication. Different from `child_process` (separate Node processes) and `cluster` (load-balanced multi-process HTTP). Real-world use is rarer than docs suggest — most "parallelism" needs are I/O-bound and async/await handles them.

**Native addons** via N-API (`node-addon-api`) provide ABI stability across Node versions. Older addons using V8 C++ APIs directly break with every Node major. Most ecosystem packages have migrated.

## Production Notes

**ESM/CJS migration.**
- Libraries should ship dual builds (ESM + CJS) until at least Node 22 LTS is the floor everywhere.
- `require(esm)` in Node 22+ removes the largest sharp edge but is not universally available yet.
- `"exports"` field in `package.json` is mandatory for proper resolution of subpath imports.

**LTS cadence.** Even-numbered majors go LTS in October, odd-numbered are non-LTS development versions. Node 18 LTS ended 2025-04; Node 20 LTS active maintenance through 2026-04; Node 22 LTS through 2027-04. Plan upgrades on this cycle.

**Native crypto and HTTP/2.** `node:crypto` is the canonical primitives source; `node:https` uses TLS 1.3 by default; HTTP/2 is supported but has subtle stream-management gotchas. Most production HTTP servers use a battle-tested framework (Fastify, Express 5, Hono) rather than raw `node:http`.

**Worker_threads at scale.** Each worker has its own V8 isolate, costing ~5–20 MB RSS. CPU-bound work parallelizes well; IPC-heavy work loses to the serialization cost. `piscina` is the standard worker pool library.

**Memory profile.** Node 20+ defaults to roughly 25% of system memory as old-generation heap. `--max-old-space-size=<mb>` is the lever. Heap dumps via `--heap-snapshot-signal=SIGUSR2` or `node --inspect`.

**Permission model.** `--permission --allow-fs-read=/app --allow-net=api.example.com` exists but is experimental and breaks many npm packages that assume unrestricted FS. Not yet production-recommended for general use.

**Single-executable applications.** `node --experimental-sea-config` produces self-contained binaries (~80 MB) embedding your script. Useful for CLI distribution.

## When to Use / When Not

**Use when:**
- You need the widest ecosystem and the safest default — npm is still the largest package registry by orders of magnitude.
- You're building HTTP services, CLI tools, build tooling, or backend-for-frontend layers.
- You need long-term stability and predictable LTS upgrades.
- You're hiring at scale — Node skills are universal.

**Avoid when:**
- Cold start matters more than throughput (Bun, Deno, or serverless-specific runtimes win).
- You want sandboxed-by-default security without configuring it yourself (Deno wins).
- You need maximum install/test speed and your team can absorb runtime diversity (Bun wins).

## Alternatives

- [oven-sh/bun](../oven-sh/bun.md) — JSC-based, faster install/start, narrower ecosystem.
- [denoland/deno](../denoland/deno.md) — V8 + Rust, secure-by-default, native TS, npm compat in 2.x.
- **Cloudflare Workers** — V8 isolates without Node compat (workerd offers Node compat as opt-in).

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.10 | 2013-03 | First widely deployed Node. |
| 4.0 | 2015-09 | Post-io.js merge, V8 4.5. |
| 8.0 | 2017-05 | `async_hooks` (experimental), N-API. |
| 10.0 | 2018-04 | Stable HTTP/2, experimental ESM. |
| 12.0 | 2019-04 | Workers stable, V8 7.4. |
| 14.0 | 2020-04 | AsyncLocalStorage stable. |
| 16.0 | 2021-04 | Apple Silicon native, V8 9.0. |
| 18.0 | 2022-04 | Built-in `fetch`, test runner, watch mode[^3]. |
| 20.0 | 2023-04 | Permission model (experimental), `node:test` improvements. |
| 22.0 | 2024-04 | `require(esm)`, WebSocket client, `--run`[^4]. |
| 24.0 | 2025-04 | Default to undici v7, V8 13+. |

## References

[^1]: Ryan Dahl, "Node.js — JSConf 2009". https://nodejs.org/en/about
[^2]: Bert Belder et al., "The Node.js Event Loop". https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
[^3]: Node.js team, "Node.js 18". https://nodejs.org/en/blog/announcements/v18-release-announce
[^4]: Node.js team, "Node.js 22". https://nodejs.org/en/blog/announcements/v22-release-announce
