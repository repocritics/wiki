# evanw/esbuild

> An extremely fast JavaScript and CSS bundler and minifier, written in Go.

[GitHub repo](https://github.com/evanw/esbuild) ·
[Official website](https://esbuild.github.io/) ·
[License: MIT](https://github.com/evanw/esbuild/blob/main/LICENSE.md)

## Overview

esbuild is a bundler, transpiler, and minifier for JavaScript, TypeScript, JSX, and CSS, written in Go and first released by Evan Wallace (co-founder of Figma) in 2020[^1]. Its explicit goal is build-tool performance: the project claims to be 10-100x faster than webpack/Rollup/Parcel by combining a compiled language, aggressive parallelism across CPU cores, and a design that avoids intermediate caches[^2]. In practice it is fast enough that most projects do not need a persistent build cache at all — a cold build is already sub-second for typical app sizes.

The defining tradeoff is scope. esbuild deliberately does *not* try to be a complete replacement for the JavaScript toolchain. It does no type checking (it strips TypeScript types without verifying them), its plugin API is intentionally narrow, and it does not implement every micro-optimization that Terser or the TypeScript compiler perform. It is a fast primitive that other tools build on, rather than a batteries-included framework. This is why it more often appears *inside* other tools than as a project's top-level build system.

That embedding is the real story of esbuild's adoption. Vite uses it for dependency pre-bundling and TS/JSX transforms during dev; frameworks and test runners (Vitest, tsup, and many others) wrap it. Its author has repeatedly stated the project is feature-complete and in maintenance mode by design[^3], which shapes how you should depend on it: stable and fast, but not a moving target that will grow to cover new use cases.

## Getting Started

```bash
npm install --save-exact --save-dev esbuild
```

```bash
# CLI: bundle an app entry point to a single minified file
npx esbuild src/app.tsx --bundle --minify --sourcemap \
  --target=es2020 --outfile=dist/app.js
```

```js
// build.mjs — the JS API, which is how most tooling drives esbuild
import * as esbuild from "esbuild";

await esbuild.build({
  entryPoints: ["src/app.tsx"],
  bundle: true,
  minify: true,
  sourcemap: true,
  target: ["es2020"],
  format: "esm",
  outfile: "dist/app.js",
});
```

## Architecture / How It Works

esbuild is a single statically-linked Go binary. The npm package is a thin JS wrapper that downloads the correct platform binary and talks to it over stdin/stdout via a small protocol; the JS API you call is marshaling requests to that child process. This is why esbuild is fast to *invoke* from Node but why plugins have a latency cost (see Production Notes).

The pipeline is a conventional bundler pipeline, but parallelized hard:

1. **Parse** — each input file is lexed and parsed into an AST. Parsing runs concurrently across a goroutine per file, saturating all CPU cores. The parser is hand-written, not generated, and handles JS, TS, JSX, and CSS.
2. **Link** — modules are resolved, scopes are analyzed, and tree shaking marks live bindings. esbuild does symbol-level dead-code elimination based on ESM's static structure.
3. **Print** — the surviving AST is printed, with minification (identifier renaming, whitespace removal, syntax lowering) folded into the same pass rather than done as a separate step.

Key design decisions that produce the speed: everything is in one process with shared memory (no serializing ASTs between plugins/loaders as in JS-based bundlers), Go's parallelism is used directly, and the AST representation is compact. Minification is not a bolt-on Terser pass — it is integrated, which is why esbuild minifies quickly but does not match Terser's most aggressive size reductions.

The **plugin API** runs `onResolve` and `onLoad` callbacks. When a plugin is JS, each callback is a round-trip from the Go process back into Node, which serializes and adds latency proportional to how many files the plugin touches. There is no AST-transform hook — plugins operate on file contents as strings, not on esbuild's internal AST. This is a hard boundary that keeps the core fast but limits what plugins can do.

## Production Notes

**TypeScript type checking is not performed.** esbuild strips types syntactically and never runs the type checker. You must run `tsc --noEmit` separately in CI; a build that passes esbuild can still be type-broken. This is documented behavior, but teams migrating from `ts-loader`/`ts-jest` are repeatedly surprised by it.

**`isolatedModules` semantics apply.** Because esbuild transpiles each file independently without whole-program type information, it cannot resolve whether an import is a type or a value. Older TS patterns (`import { SomeType }` used only as a type, `const enum`, namespace merging) can break. Use `import type`, and set `"isolatedModules": true` in tsconfig so `tsc` flags the same issues esbuild would trip on.

**Version pinning matters.** esbuild's own guidance is to install with an exact version (`--save-exact`). The npm package's postinstall pulls a platform-specific binary, and a version mismatch between the JS wrapper and the binary (common in monorepos or with hoisting, or behind restrictive npm proxies that block the optional platform packages) produces a runtime "host version does not match binary version" error. This is the single most common operational failure.

**Output size vs. Terser/Rollup.** esbuild's minifier is fast but slightly less aggressive; bundles are often a few percent larger than a Rollup + Terser pipeline. For most apps this is irrelevant; for size-critical libraries shipped to npm, Rollup remains the more common choice for the final artifact.

**Plugin latency.** A JS plugin that intercepts many files adds a Node<->Go round-trip per file and can erase esbuild's speed advantage. Native performance is only retained when plugins are cheap or few. Heavy transform needs (e.g., a full PostCSS pipeline) partly defeat the point.

**Maintenance-mode expectations.** The project intentionally rejects most feature requests as out of scope and is stable rather than evolving[^3]. Depend on it for what it does today; do not plan around features landing later. Node compatibility, decorators, and some legacy CSS features have historically been non-goals or added slowly.

## When to Use / When Not

**Use when:**
- You need fast TS/JSX transpilation or bundling as a build primitive (dev servers, test runners, library prepublish steps).
- You want a single fast binary with no plugin ecosystem to assemble.
- You are building a tool and want a bundler to embed rather than a framework to adopt.
- Build speed is the dominant cost and you can run type checking separately.

**Avoid when:**
- You need the smallest possible output for a published library — Rollup + Terser still edges it out on size.
- You rely on a rich plugin ecosystem for complex transforms (heavy PostCSS, Vue SFC, exotic asset pipelines) — Vite/Rollup ecosystems are deeper.
- You want the bundler itself to type-check — it never will.
- You need features the author has declared out of scope; the project will not grow to meet you.

## Alternatives

- vitejs/vite — use instead when you want a full dev-server + HMR + plugin ecosystem; Vite uses esbuild internally for transforms.
- rollup/rollup — use instead when publishing a library and final bundle size / output control matters most.
- swc-project/swc — use instead when you want a Rust-based transpiler with a native Node API and deep AST-level plugin support.
- privatenumber/tsup — use instead when you want an esbuild wrapper with sane library-publishing defaults (dts, multiple formats).
- webpack/webpack — use instead when you need maximum plugin/loader coverage and legacy ecosystem compatibility over speed.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2020-01 | First public release; Go bundler for JS/TS/JSX[^1]. |
| 0.8 | 2020-11 | Plugin API (`onResolve`/`onLoad`) introduced. |
| 0.11 | 2021-03 | Incremental build and rebuild API. |
| 0.14 | 2021-12 | CSS handling and content-type improvements. |
| 0.17 | 2023-01 | Reworked incremental/watch/serve context API. |
| 0.19 | 2023-07 | Continued CSS and target-lowering work. |
| 0.21 | 2024-05 | Ongoing syntax-lowering and compatibility fixes. |
| 0.25 | 2025 | Latest 0.x line; still pre-1.0, API stable in practice[^3]. |

## References

[^1]: Evan Wallace, "esbuild: An extremely fast JavaScript bundler." Project site and origin. https://esbuild.github.io/
[^2]: esbuild documentation, "Why is esbuild fast?" — architecture and benchmark rationale. https://esbuild.github.io/faq/#why-is-esbuild-fast
[^3]: esbuild FAQ, "Production readiness" and roadmap / scope statements by the author. https://esbuild.github.io/faq/#production-readiness

## Tags

javascript, typescript, bundler, minifier, build-tool, go, jsx, css, esm, compiler, web-tooling
