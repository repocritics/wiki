# denoland/deno

> A modern runtime for JavaScript and TypeScript — secure by default, web-standards-first.

[GitHub repo](https://github.com/denoland/deno) ·
[Official website](https://deno.com) ·
[License: MIT](https://github.com/denoland/deno/blob/main/LICENSE.md)

## Overview

Deno is a JavaScript and TypeScript runtime created in 2018 by Ryan Dahl, also the original author of Node.js[^1]. Dahl's 2018 talk "10 Things I Regret About Node.js" framed Deno as a list of fixes: explicit permissions for filesystem / network / env access, native TypeScript without a build step, ES modules as the only module system, web-standard APIs (`fetch`, `URL`, `Web Streams`) as the primary API surface, and a single self-contained binary.

Deno 2.0 (2024-10) was the biggest course correction in the project's history[^2]. The original "no npm" stance had become a major adoption blocker; 2.0 shipped first-class Node and npm compatibility (`npm:` and `node:` import specifiers, `package.json` support, `node_modules` resolution), launched the **JSR** registry as a TypeScript-first alternative to npm, and made workspaces stable.

Deno is implemented in Rust with V8 as the JavaScript engine and Tokio as the async runtime[^3]. This is the same Rust + V8 architecture other projects (e.g., `swc`) use, distinct from Node (C++) and Bun (Zig + JSC).

## Getting Started

```bash
curl -fsSL https://deno.land/install.sh | sh
```

Run a TypeScript file directly:

```ts
// server.ts
Deno.serve((req) => new Response("Hello from Deno"))
```

```bash
deno run --allow-net server.ts
```

Initialize a project:

```bash
deno init my-app
cd my-app
deno task dev
```

Import npm and JSR packages:

```ts
import express from "npm:express@4"
import { encodeHex } from "jsr:@std/encoding/hex"
import { readFile } from "node:fs/promises"
```

## Architecture / How It Works

Deno's permissions model is the most distinctive design choice[^4]. By default, a Deno program cannot read the filesystem, make network requests, or read environment variables. Flags grant explicit capabilities:

```bash
deno run --allow-read=./data --allow-net=api.example.com:443 app.ts
```

This is genuinely useful for running untrusted code (CI runners, plugin sandboxes), and modestly useful for production servers as a defense-in-depth layer. It is also frequently disabled with `-A` (allow all) in practice, which collapses much of the security pitch.

**Module resolution** uses three specifier styles:
- **HTTPS** imports: `import x from "https://deno.land/x/foo@1.0/mod.ts"` — fetched once, cached. The original Deno approach, now discouraged for production due to centralization risk.
- **npm:** imports: `import express from "npm:express@4"` — resolved through Deno's npm-compatibility layer.
- **JSR:** imports: `import { x } from "jsr:@scope/pkg"` — TypeScript-first, versioned, supports types without a build step.

**JSR** (jsr.io) is Deno's package registry, launched in 2024. Unlike npm, JSR is TypeScript-first — packages publish their source directly, the registry generates `.d.ts` and provenance signatures, and dual-publishes to npm with `dnt` for cross-runtime use.

**Built-in tooling**: `deno fmt`, `deno lint`, `deno test`, `deno bench`, `deno doc`. The `deno task` runner uses scripts defined in `deno.json` instead of `package.json`'s `scripts`. There is no separate bundler step required for distribution; `deno compile` produces self-contained executables.

**Deno KV** (`Deno.openKv()`) is a built-in key-value store with optional cloud-backed persistence via Deno Deploy. Useful for small stateful apps; not a replacement for Postgres or Redis.

## Production Notes

**Node compat caveats.** The compatibility layer handles most real-world Node packages, but the boundary is fuzzy. Packages with native addons, certain `vm` usages, or assumptions about `process.cwd()` behavior can break. As of 2025, the npm-compatibility issue tracker is the canonical reference[^5].

**Permission UX.** `-A` is the default for most local development. Production deployments should narrow with `--allow-net=specific.host`, `--allow-read=/app`, etc. The permission system does not protect against side-channel attacks; it's defense-in-depth.

**Module URLs are out.** HTTPS imports were the original signature feature but had three problems: centralization (deno.land/x became a SPOF), cache invalidation (lock files were retrofitted), and tooling (transitive dependency resolution was harder than npm's). The recommended path in Deno 2 is `jsr:` + `npm:`.

**Deno Deploy.** Deno's edge serverless platform runs Deno code on Cloudflare-like edge POPs. KV is integrated; cold starts are fast. Pricing model is similar to Cloudflare Workers. Lock-in is real but escape via `deno compile` + container is straightforward.

**TypeScript performance.** Deno's TS handling is faster than `tsc` because it uses `swc` to strip types and only invokes `tsc` on demand for diagnostics. Large codebases still benefit from `deno check` in CI rather than relying on inline type checking.

**Workspaces.** Stable as of Deno 2. `deno.json` supports a `workspace` field listing member directories. Behavior parallels pnpm workspaces.

## When to Use / When Not

**Use when:**
- You're starting a new TypeScript service and want to skip the build / tsconfig / package.json setup tax.
- You want web-standard APIs (`fetch`, `Web Streams`) as your primary API surface.
- You need a sandboxed runtime for untrusted code (plugins, eval-style features).
- You're targeting Deno Deploy or a similar edge runtime where Deno is first-class.

**Avoid when:**
- You depend on a Node native addon ecosystem that hasn't been validated against Deno (worker_threads-heavy code, native crypto extensions).
- You're publishing a library and need maximum runtime portability — publish to npm, dual-publish to JSR optionally.
- You want a binary lockfile and dramatically faster install (Bun fits better there).

## Alternatives

- [nodejs/node](../nodejs/node.md) — broadest ecosystem, no permission model, longer build chain.
- [oven-sh/bun](../oven-sh/bun.md) — JSC-based, faster install/start, less mature security model.
- **Cloudflare Workers** — V8 isolates, web-standards-first like Deno, but stricter runtime constraints.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2018-08 | Initial release, "Deno: A new way to JavaScript"[^1]. |
| 1.0 | 2020-05 | Stable, permission model. |
| 1.18 | 2022-01 | `npm:` specifier preview. |
| 1.25 | 2022-09 | `npm:` specifier stable, Node compat layer. |
| 1.32 | 2023-03 | Deno KV preview. |
| 1.40 | 2024-01 | Workspaces preview. |
| 2.0 | 2024-10 | Stable Node compat, JSR, workspaces[^2]. |
| 2.1 | 2024-11 | Wasm imports, frozen lockfile improvements. |
| 2.2 | 2025-02 | Built-in OTLP tracing, performance work. |

## References

[^1]: Ryan Dahl, "10 Things I Regret About Node.js" — JSConf EU 2018. https://www.youtube.com/watch?v=M3BM9TB-8yA
[^2]: Deno Team, "Deno 2 is here" — 2024-10-09. https://deno.com/blog/v2.0
[^3]: Deno manual, "Internal Details". https://docs.deno.com/runtime/reference/web_platform_apis/
[^4]: Deno manual, "Permissions". https://docs.deno.com/runtime/fundamentals/security/
[^5]: Deno repo, "Node API compat". https://docs.deno.com/runtime/reference/node/
