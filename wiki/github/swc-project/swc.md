# swc-project/swc

> A Rust-based TypeScript/JavaScript compiler toolchain — a Babel/Terser replacement that runs 10–70× faster and is embedded inside most of the modern JS toolchain.

[GitHub repo](https://github.com/swc-project/swc) ·
[Official website](https://swc.rs) ·
[License: Apache-2.0](https://github.com/swc-project/swc/blob/main/LICENSE)

## Overview

SWC ("Speedy Web Compiler") is a compiler platform for the web, written in Rust, started by DongYoon Kang (kdy1) in 2017[^1]. Its job is the unglamorous middle of a JS build: parse TypeScript/JSX/modern ECMAScript into an AST, apply transforms (down-leveling syntax, stripping types, JSX, decorators, minification), and emit JavaScript plus source maps. It is simultaneously a set of Rust crates (`swc_ecma_parser`, `swc_ecma_transforms`, `swc_ecma_minifier`, …) and a set of npm packages (`@swc/core`, `@swc/cli`, `@swc/jest`) exposing them to Node via native bindings.

The reason SWC matters is less that people invoke it directly and more that it is the compiler underneath things they already use. Next.js replaced Babel with SWC in version 12 (2021)[^2]; Deno, Parcel 2, Vitest, Rspack, Turbopack, Rollup's `@rollup/plugin-swc`, and Nx all use SWC crates for parsing or transformation. That embedding is the defining tension: SWC's Rust API is where the real power lives, but the vast majority of users touch it only through a host tool that pins a specific SWC version, so SWC's own versioning discipline (and its plugin ABI churn) becomes their problem indirectly.

Like Babel and unlike `tsc`, SWC **does not type-check** — it strips and rewrites, trusting your types are already valid. This is a deliberate speed tradeoff, not a missing feature: type-checking is the slow part, so SWC leaves it to `tsc --noEmit` in a parallel step.

## Getting Started

```bash
npm i -D @swc/core @swc/cli
```

Configure via `.swcrc` and compile:

```json
{
  "$schema": "https://swc.rs/schema.json",
  "jsc": {
    "parser": { "syntax": "typescript", "tsx": true },
    "target": "es2022",
    "transform": { "react": { "runtime": "automatic" } }
  },
  "module": { "type": "es6" }
}
```

```bash
npx swc src -d dist          # compile a directory
```

Programmatic use from Node:

```js
import { transform } from "@swc/core";

const { code, map } = await transform(source, {
  filename: "input.tsx",
  jsc: { parser: { syntax: "typescript", tsx: true }, target: "es2022" },
  sourceMaps: true,
});
```

## Architecture / How It Works

SWC is a monorepo of ~90 Rust crates. The pipeline for a file is: `swc_ecma_parser` (a hand-written recursive-descent lexer + parser producing a typed AST) → one or more `swc_ecma_transforms_*` passes (a `Fold`/`VisitMut` visitor tree) → `swc_ecma_codegen` (emits source + source map). `swc_ecma_minifier` is a separate transform aiming for Terser output parity, and `swc_bundler` provides bundling used in some downstream tools.

Two design choices explain most of the speed. First, the AST and visitor machinery are built to run passes in a single tree walk where possible and to avoid re-allocating strings (`swc_atoms` interns identifiers). Second, the Node bindings use napi-rs, so `@swc/core` calls into native code with no serialization boundary — the JS AST is never materialized unless you ask for it. Parallelism across files is left to the host (Jest, Next.js) which shards work over a thread pool.

The most consequential architectural decision for users is the **Wasm plugin system**. Custom transforms are written as Rust crates compiled to `wasm32-wasi`, loaded at runtime via Wasmer. This is what lets Next.js and others accept community transforms without recompiling SWC. The catch is that the plugin and the host both depend on `swc_core`, and the AST types cross the Wasm boundary via a serialized ABI that is **not stable across `swc_core` major/minor bumps**[^3]. A plugin compiled against one `swc_core` version will refuse to load (or misbehave) against a mismatched host, which is the single largest source of SWC-related breakage in practice.

## Production Notes

**Plugin/version coupling is the footgun.** If you use SWC Wasm plugins (`@swc/plugin-styled-components`, `@swc/plugin-emotion`, custom transforms), the plugin's `swc_core` version must be compatible with the `@swc/core` your tool ships. Upgrading Next.js can silently pull a new SWC and break a plugin that pinned an older ABI; the failure mode is a cryptic load error, not a clear version diagnostic. Pin plugin versions deliberately and treat the plugin ecosystem as tightly coupled to your bundler's release cadence[^3].

**Decorators and edge TS features.** `emitDecoratorMetadata` (used heavily by NestJS/TypeORM) and legacy decorators have historically had parity gaps and correctness bugs versus `tsc`. Behavior has improved, but teams on decorator-heavy stacks should verify runtime behavior rather than assume drop-in equivalence.

**No type-checking means no type errors in your build.** SWC will happily compile code that `tsc` would reject. You need a separate `tsc --noEmit` (or `isolatedDeclarations`) step in CI, and you must write code under `isolatedModules` assumptions — const enums and certain type-only re-exports need care because SWC compiles file-by-file without whole-program type information.

**Minifier parity.** `swc_ecma_minifier` targets Terser output but is not byte-identical; for most apps this is invisible, but if you depend on Terser-specific mangling or have been bitten by a minifier miscompilation before, benchmark output size and correctness before switching your production minify step off Terser/esbuild.

**Source maps and `.swcrc` scope.** Nested `.swcrc` resolution and the interaction between `jsc.target`, `env.targets`, and browserslist can produce surprising down-leveling. Set one clearly and avoid mixing `jsc.target` with `env` in the same config.

## When to Use / When Not

**Use when:**
- You want Babel/Terser semantics at a fraction of the build time, invoked from Node or a bundler.
- You're building tooling in Rust and need a fast, correct JS/TS parser and transformer (this is SWC's strongest, most stable surface).
- You need fast test transforms — `@swc/jest` is a common drop-in for `babel-jest`/`ts-jest`.

**Avoid when:**
- You need type-checking as part of compilation — that's `tsc`, not SWC.
- Your build depends on a large, exotic Babel plugin ecosystem with no SWC equivalent; porting to a Wasm plugin is nontrivial.
- You need long-term ABI stability for custom transforms — the Wasm plugin ABI still moves, so maintaining plugins is ongoing work.

## Alternatives

- evanw/esbuild — Go-based transform+bundler; comparable speed, includes bundling out of the box, smaller plugin surface. Use instead when you want one tool that also bundles.
- oxc-project/oxc — newer Rust linter/parser/transformer suite chasing SWC's niche with a focus on correctness and a single toolchain. Use when evaluating a fresher Rust alternative.
- babel/babel — the mature, plugin-rich reference implementation. Use when you depend on Babel-only plugins or need maximum spec-edge fidelity over speed.
- microsoft/TypeScript — `tsc` is the only one here that actually type-checks. Use it (in parallel) for type safety; SWC does not replace it.
- terser/terser — the JS minifier SWC's minifier targets. Use when you need reference-grade, battle-tested minification output.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-12 | Repository created; project started by kdy1[^1]. |
| adopted by Next.js | 2021-10 | Next.js 12 replaces Babel with SWC as default compiler[^2]. |
| Wasm plugins | 2022 | Runtime Rust→Wasm plugin system introduced; ABI not stable across `swc_core` versions[^3]. |
| ongoing | 2026-07 | Actively developed; ~34k stars, MSRV Rust 1.73, Apache-2.0. Community-maintained by volunteers[^4]. |

## References

[^1]: SWC project and rustdoc — DongYoon Kang (kdy1). https://swc.rs and https://rustdoc.swc.rs/swc/
[^2]: Next.js 12 blog, adoption of the SWC compiler (2021-10). https://nextjs.org/blog/next-12
[^3]: SWC docs, "Using swc plugins" — Wasm plugin ABI / `swc_core` version compatibility. https://swc.rs/docs/plugin/ecmascript/getting-started
[^4]: SWC README and team page — community-driven, maintained by volunteers. https://swc.rs/docs/team

## Tags

rust, javascript, typescript, compiler, transpiler, parser, minifier, build-tooling, babel-alternative, wasm-plugins, ecmascript
