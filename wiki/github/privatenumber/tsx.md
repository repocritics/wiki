# privatenumber/tsx

> Run TypeScript (and modern ESM) directly in Node.js by transpiling on the fly with esbuild — no type-checking, no build step.

[GitHub repo](https://github.com/privatenumber/tsx) ·
[Official website](https://tsx.hirok.io) ·
[License: MIT](https://github.com/privatenumber/tsx/blob/master/LICENSE)

## Overview

tsx ("TypeScript Execute") is a CLI and Node.js loader that lets you run `.ts`,
`.tsx`, `.mts`, and `.cts` files as if Node understood TypeScript natively. It
was created by Hiroki Osame (privatenumber) in 2022 and originally lived under
the `esbuild-kit` organization before being consolidated. As of 2026 it is one
of the most-used ways to execute TypeScript in Node, sitting alongside ts-node
as the de facto answer to "how do I just run this `.ts` file." The repo is
actively maintained — 12k stars, 244 forks, with commits landing the same week
as this writing.

The defining tradeoff is deliberate and load-bearing: **tsx transpiles, it does
not type-check.** It hands your source to esbuild, which strips types and
rewrites syntax at high speed but performs zero semantic analysis. This makes
startup fast and the tool nearly config-free, but it means a file with type
errors runs happily to completion. tsx is an execution tool, not a compiler;
type safety is expected to come from a separate `tsc --noEmit` in your editor
or CI. Treating tsx as a replacement for `tsc` is the single most common
misunderstanding.

The second tension is Node integration. Running TypeScript "natively" requires
hooking Node's module system, and those hooks are version-sensitive. Much of
tsx's engineering is spent papering over differences between Node's ESM loader
API versions and CJS `require` interception so that one command behaves
consistently across Node 18 through 24.

## Getting Started

```bash
npm install -D tsx
```

```bash
npx tsx ./src/index.ts          # run a file directly
npx tsx watch ./src/server.ts   # restart on file change
npx tsx                         # REPL with TypeScript support
node --import tsx ./src/index.ts # as a loader, keeping Node's own flags
```

```ts
// src/index.ts — no tsconfig, no build required
import { readFile } from "node:fs/promises";

const pkg: { name: string } = JSON.parse(
  await readFile("./package.json", "utf8"),
);
console.log(`Running ${pkg.name}`); // top-level await just works
```

## Architecture / How It Works

tsx is a thin orchestration layer over **esbuild**. Its real work is installing
Node module hooks that intercept resolution and loading of TypeScript files and
route their contents through esbuild's transform API before Node evaluates them.

- **ESM path.** On modern Node (20.6+), tsx registers via `module.register()`,
  which runs loader hooks in a worker thread. On older Node it falls back to the
  now-deprecated `--loader` / `--experimental-loader` flag. The hooks resolve
  extensionless and `.js`-specified imports to their `.ts` sources and transform
  on load.
- **CJS path.** For `require()`, tsx patches `require.extensions` (and Module
  internals) so that requiring a `.ts` file transpiles it synchronously. This is
  how a single tool covers both module systems in one process.
- **esbuild transform.** Types are erased and syntax lowered to match the
  running Node; fast, but esbuild's TS support has known gaps (see below). tsx
  injects source maps so stack traces point at original `.ts` lines.
- **tsconfig awareness.** tsx reads `tsconfig.json` for a subset of options that
  matter at runtime — notably `paths` alias resolution, `jsx`/`jsxFactory`, and
  `target` — but it does not honor type-level configuration, because it never
  type-checks.

The coupling to esbuild is total: tsx inherits esbuild's speed, its install
footprint (a platform-specific native binary, tens of MB), and its limitations.
tsx's own code is mostly the Node-integration glue esbuild does not provide.

## Production Notes

- **No type errors, ever.** A build that would fail `tsc` runs fine under tsx.
  You must run `tsc --noEmit` separately in CI. Teams that skip this ship type
  errors to production. This is by design, not a bug.
- **esbuild's TS gaps are tsx's gaps.** `emitDecoratorMetadata` is not supported
  by esbuild, so frameworks that rely on decorator metadata at runtime
  (older NestJS, TypeORM, `reflect-metadata` DI patterns) can misbehave under
  tsx. `const enum` and some ambient constructs also have caveats.
- **Node version sensitivity.** Because tsx rides Node's loader hooks, behavior
  can shift across Node majors — the ESM hook API changed meaningfully around
  Node 18.19 / 20.6, and deprecation warnings for `--loader` appear on some
  versions. Pin your Node version in CI and expect loader edge cases after Node
  upgrades. The bundled esbuild native binary also makes tsx heavier than a
  pure-JS tool, which matters for slim images and cold CI caches.
- **Not for shipping libraries.** tsx runs source; it does not produce a `dist/`.
  For publishing a package you still need a real build (tsc, tsup, unbuild,
  rollup). tsx is for scripts, dev servers, tests, and tooling entry points.
- **`tsx watch` restarts the process** rather than hot-reloading module state —
  fine for servers, but in-memory state is lost on each change.
- **Native Node overlap.** Node 22.6+ can strip types behind
  `--experimental-strip-types`, and Node 23.6+ strips types by default. For
  simple type-annotation-only files this reduces the need for tsx; tsx still
  wins for `paths`, JSX, syntax lowering to older Node, and enum/namespace
  features that Node's stripper rejects.

## When to Use / When Not

**Use when:**
- You want to run a `.ts` script or dev server with zero build config.
- You need fast iteration (`tsx watch`) during development.
- You run TypeScript test files, seed scripts, or CLI entry points.
- You need `tsconfig` `paths` resolution and JSX without wiring a bundler.

**Avoid when:**
- You expect it to catch type errors — it never will; pair it with `tsc`.
- You depend on `emitDecoratorMetadata` or other esbuild-unsupported TS
  features.
- You are producing a distributable build artifact (use tsc/tsup/rollup).
- You are on a runtime that already executes TypeScript (Bun, Deno) or can rely
  on Node's native type stripping for simple cases.

## Alternatives

- TypeStrong/ts-node — the original Node TS runner; can actually type-check
  (or run swc for speed). Use when you want type errors to halt execution.
- swc-project/swc (`@swc-node/register`) — Rust-based transpile-only loader;
  use when you want a ts-node-style loader on the swc toolchain.
- oven-sh/bun — a runtime that executes TypeScript natively; use when you can
  adopt a non-Node runtime and want no loader layer at all.
- denoland/deno — runs TS natively with its own toolchain; use for greenfield
  projects not tied to the npm/Node module system.
- Node.js native type stripping — built-in since Node 23.6; use when your files
  are annotation-only and you want zero dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2022-05 | Initial release under the esbuild-kit org; esbuild-powered TS execution and watch mode. |
| 3.x | 2023 | Consolidation and loader improvements; broadened Node ESM support. |
| 4.0 | 2023-10 | Major release; reworked module loading, moved to the `privatenumber/tsx` repo, dropped older loader assumptions. |
| 4.x | 2024–2026 | Ongoing support for new Node loader-hook APIs (Node 20.6 `module.register`, Node 22/23 changes) and esbuild upgrades. |

## References

[^1]: tsx documentation and getting-started guide. https://tsx.is (redirects from https://tsx.hirok.io)
[^2]: tsx repository README and releases. https://github.com/privatenumber/tsx/releases
[^3]: esbuild — the transform engine tsx depends on; TypeScript caveats. https://esbuild.github.io/content-types/#typescript
[^4]: Node.js module customization hooks (`module.register`). https://nodejs.org/api/module.html#customization-hooks
[^5]: Node.js native TypeScript support (type stripping). https://nodejs.org/api/typescript.html

## Tags

typescript, node, cli, runtime, esbuild, esm, loader, watch, dev-tools, transpiler
