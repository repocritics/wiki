# oven-sh/bun

> Incredibly fast JavaScript runtime, bundler, transpiler, and package manager — all in one.

[GitHub repo](https://github.com/oven-sh/bun) ·
[Official website](https://bun.sh) ·
[License: MIT](https://github.com/oven-sh/bun/blob/main/LICENSE.md)

## Overview

Bun is a JavaScript/TypeScript runtime, bundler, transpiler, package manager, and test runner shipped as a single Zig binary[^1]. It uses **JavaScriptCore** (Apple's engine, the same one in Safari) rather than V8, implements much of Node.js's API surface for compatibility, and adds Bun-native APIs (`Bun.serve`, `Bun.file`, `Bun.sql`, `Bun.spawn`).

The pitch is throughput: cold start, install speed, HTTP server requests/sec, and test execution all measure 2–10× faster than Node + ecosystem equivalents on standard benchmarks[^2]. Whether that translates to your specific workload depends heavily on how much your code touches Node-compatibility shims versus Bun-native APIs.

Bun 1.0 (2024-09) marked the production-ready milestone[^3]. The 1.x cycle filled Node API gaps, improved Windows support (initially much weaker than macOS/Linux), and added the `bun sql` Postgres driver. As of 2025 Bun is in production at a meaningful number of companies, particularly for HTTP servers, internal tools, and CI pipelines, though Node remains the safer default for libraries that need to work everywhere.

## Getting Started

```bash
curl -fsSL https://bun.sh/install | bash
```

Initialize a project:

```bash
bun init
bun add express
bun run index.ts
```

A minimal HTTP server using the native API:

```ts
Bun.serve({
  port: 3000,
  fetch(req) {
    return new Response("Hello from Bun")
  }
})
```

TypeScript and JSX run directly without configuration:

```bash
bun run app.tsx
```

Package management:

```bash
bun install   # ~20× faster than npm in cold-cache benchmarks
bun add pkg
bun remove pkg
```

## Architecture / How It Works

Bun is written in Zig with a JavaScriptCore embedding[^1]. JSC is widely regarded as having better cold-start and lower memory baseline than V8, at the cost of slightly lower peak throughput on long-running benchmarks. Bun's IO is handled via a custom event loop (not libuv).

The bundler / transpiler is the same code path that handles `bun run`: source files are parsed by Bun's TypeScript-aware parser, transformed, and either executed directly or written to disk. There is no separate `tsc`-equivalent type check — Bun strips types but does not check them. Type checking is still delegated to `tsc --noEmit`.

The package manager uses a binary lockfile (`bun.lockb`) rather than the JSON `package-lock.json` used by npm. This is faster to read/write but cannot be reviewed in pull requests without `bun.lockb --print`[^4]. Bun 1.2+ added an optional textual lockfile (`bun.lock`) for compatibility.

Bun-native APIs vs Node compat:
- **Bun.serve** — native HTTP server, ~3× faster than `http.createServer`.
- **Bun.file** — file API with lazy I/O; reading metadata doesn't read content.
- **Bun.sql** — built-in Postgres driver.
- **Bun.spawn** — child process API; cleaner stream handling than `child_process`.
- **node:** prefix — `node:fs`, `node:http`, etc. work as Node compat shims.

## Production Notes

**Node API gaps.** The compatibility surface is high (>95% of standard library by usage), but specific holes still bite[^5]:
- `vm` module is partial — `node-jest` and similar tools may break.
- `worker_threads` is supported but with subtle perf characteristics differing from Node.
- Native addons via N-API: supported, but ABI compatibility means slightly older addon versions may fail.
- `cluster` module: supported but the underlying socket sharing differs.

**Memory profile.** Bun typically uses 30–60% less RSS than Node for equivalent workloads. JSC's generational GC has different pause characteristics than V8 — long-running services should profile before assuming Bun's defaults match Node's.

**Windows support.** Originally weak (1.0 shipped with significant Windows gaps); 1.2+ is meaningfully better but still trails macOS/Linux. CI on Windows is the most common surface for issues.

**Lockfile diffing.** `bun.lockb` is binary. Use `bun pm hash` and `bun.lock` (textual) for human-readable diffs. Most teams enable the textual lockfile in CI.

**Workspace / monorepo.** Bun workspaces are supported (`workspaces: ["packages/*"]` in `package.json`). Hoisting behavior differs subtly from pnpm — packages that rely on a flat `node_modules` may break.

**Test runner.** `bun test` is Jest-compatible at the assertion level (`expect()`, `describe`, `it`). It is dramatically faster, but does not support every Jest plugin. Snapshot, mocks, and coverage are built in.

**Bun.sql Postgres.** Production-ready as of 1.2, but the connection pool semantics are different from `pg`/`postgres.js`. Audit transaction behavior under load.

## When to Use / When Not

**Use when:**
- You need fast cold starts (CLI tools, serverless functions, CI scripts).
- You're building an internal HTTP service where Bun's native APIs simplify the code.
- Your `bun install` time is a meaningful cost in your CI.
- You want a single binary for runtime + bundler + package manager + test runner.

**Avoid when:**
- You're publishing a library and need the broadest possible compatibility — target Node and test on both.
- You depend on native addons that haven't been validated against JSC.
- You need a feature in the `vm`, `cluster`, or `worker_threads` modules with niche behavior.
- Your team isn't ready to debug runtime-level differences (memory, GC, event loop ordering).

## Alternatives

- [nodejs/node](../nodejs/node.md) — the canonical, safest default, broader ecosystem.
- [denoland/deno](../denoland/deno.md) — V8-based, secure-by-default, web-standards-first, npm compat in Deno 2.
- **Just `tsx` or `esno`** — fast TS execution on Node without committing to a new runtime.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2022-07 | Initial Jarred Sumner release[^1]. |
| 0.5 | 2023-02 | `bun test`, improved Node compat. |
| 0.7 | 2023-07 | Workspaces, `bun pm`. |
| 1.0 | 2023-09 | Production-ready[^3]. |
| 1.1 | 2024-04 | Windows preview, `bun shell`. |
| 1.2 | 2024-12 | Textual lockfile, Bun.sql Postgres GA. |
| 1.x | 2025 | Continued Node-compat fills, Bun.sql MySQL, edge runtime adapters. |

## References

[^1]: Jarred Sumner, "Bun, a new JavaScript runtime" — 2022-07. https://bun.sh/blog/bun-v0.1.0
[^2]: Bun team, "Bun benchmarks". https://bun.sh/docs/benchmarks
[^3]: Jarred Sumner, "Bun 1.0" — 2023-09-08. https://bun.sh/blog/bun-v1.0
[^4]: Bun docs, "Lockfile". https://bun.sh/docs/install/lockfile
[^5]: Bun docs, "Node.js compatibility". https://bun.sh/docs/runtime/nodejs-apis
