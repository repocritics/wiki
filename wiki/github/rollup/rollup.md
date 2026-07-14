# rollup/rollup

> An ES-module-first JavaScript bundler that popularized tree-shaking and became the standard tool for shipping libraries — and the production bundler inside Vite.

[GitHub repo](https://github.com/rollup/rollup) ·
[Official website](https://rollupjs.org) ·
[License: MIT](https://github.com/rollup/rollup/blob/master/LICENSE.md)

## Overview

Rollup is a module bundler that compiles many small JavaScript files into fewer, larger ones. It was written by Rich Harris (later the author of Svelte) and takes ES modules as its native, canonical format rather than treating them as one input among several[^1]. That single design decision — assume `import`/`export`, analyze statically, and only include what is actually reached — is what let Rollup coin and popularize "tree-shaking"[^2], and it is why Rollup output has historically been smaller and flatter than webpack's for the same code.

For most of its life the division of labour in the ecosystem was: **webpack for applications, Rollup for libraries.** Rollup emits clean ESM/CJS/UMD without a runtime wrapper or module registry, which is exactly what a package author wants and exactly what an app with code-splitting, hot reload, and hundreds of dependencies does not get for free. That framing changed when Vite adopted Rollup as its production bundler: today the largest population of Rollup users have never written a `rollup.config.js` and reach it only through Vite[^3].

The defining tension in 2026 is succession. Rollup 4 rewrote its parser in Rust for speed[^4], but the deeper performance ceiling — a single-threaded JavaScript bundling core — is being addressed by **Rolldown**, a ground-up Rust reimplementation of the Rollup API from the Vite team, intended to eventually replace both Rollup and esbuild inside Vite[^5]. Rollup remains the stable, mature, widely-depended-upon tool; Rolldown is the declared future of the same interface.

## Getting Started

```bash
npm install --save-dev rollup
# starter templates: rollup/rollup-starter-lib, rollup/rollup-starter-app
```

```js
// rollup.config.js
import resolve from '@rollup/plugin-node-resolve';
import terser from '@rollup/plugin-terser';

export default {
  input: 'src/main.js',
  output: [
    { file: 'dist/bundle.cjs',    format: 'cjs' },
    { file: 'dist/bundle.mjs',    format: 'es'  },
    { file: 'dist/bundle.umd.js', format: 'umd', name: 'MyLib' },
  ],
  plugins: [resolve(), terser()],
  external: ['react'],   // don't bundle peer deps into a library
};
```

```bash
rollup -c            # build using rollup.config.js
rollup -c -w         # watch mode
```

## Architecture / How It Works

Rollup runs in two phases. The **build** phase parses every module, resolves imports through the plugin chain, and constructs a single module graph; the **output/generate** phase performs tree-shaking, ordering, and format-specific code generation, and can run multiple times against one build (one graph → several output formats). This split is why emitting CJS + ESM + UMD from one source is cheap.

Tree-shaking is not minification. Rollup reasons about the module graph statically: it marks exports that are never imported, and — crucially — tracks whether module-level code has side effects, so that pulling one function out of a file does not drag in the rest[^2]. This works only because ES `import`/`export` are static; dynamic `require()` and computed access defeat it, which is why CommonJS dependencies must be converted (`@rollup/plugin-commonjs`) before they can be shaken.

The **plugin system** is Rollup's most consequential export. Plugins are objects of well-defined hooks (`resolveId`, `load`, `transform`, `renderChunk`, `generateBundle`, …). Vite's plugin API is a superset of Rollup's, so a large share of the plugin ecosystem is portable between them, and the `unplugin` project builds one plugin that targets Rollup, Vite, webpack, and esbuild through this shared shape[^3]. Learning Rollup's hooks is therefore leverage well beyond Rollup itself.

Since **Rollup 4 (October 2023)** the JavaScript parser is SWC's Rust implementation compiled to native binaries, replacing the pure-JS Acorn parser used through v3[^4]. This is distributed as platform-specific optional dependencies (`@rollup/rollup-linux-x64-gnu` and siblings), which is fast but introduces an npm install-time footgun (see Production Notes). The rest of the bundler core remains JavaScript.

## Production Notes

**The optional-dependency install bug.** Because native binaries ship as `optionalDependencies`, a broken or partial npm install can leave you with `Cannot find module @rollup/rollup-<platform>`. This has bitten CI reproducibly, especially with a stale `package-lock.json`, `npm ci` after switching platforms, or Docker layer caching. The reliable fix is to delete `node_modules` and `package-lock.json` and reinstall; it is a known npm lockfile issue, not a Rollup bug per se[^6].

**Library, not app, by default.** Rollup does not give you a dev server, HMR, or asset pipeline. Reaching for Rollup directly to bundle an application means re-assembling those from plugins. If you want the application experience, you almost certainly want Vite (which is Rollup underneath) rather than raw Rollup.

**CommonJS interop is a recurring source of pain.** Anything not authored as ESM needs `@rollup/plugin-commonjs`, and mixed named/default interop, dynamic `require`, and circular CJS dependencies can produce output that differs from Node's own resolution. Order matters: `@rollup/plugin-node-resolve` before `commonjs`.

**Bundle-time performance.** For large graphs Rollup is meaningfully slower than esbuild/Rust bundlers because the core is single-threaded JavaScript. This is the explicit motivation for Rolldown; if build speed on a big app is your bottleneck, that is the axis to watch rather than tuning Rollup config[^5].

**Externalization discipline for libraries.** Failing to mark peer dependencies as `external` silently inlines React (or similar) into your published package, causing duplicate-copy bugs downstream. This is the single most common library-authoring mistake with Rollup.

**Upgrade notes.** v3 (Oct 2022) dropped older Node and reworked the config/CLI surface; v4 (Oct 2023) swapped in the native parser and its optional-dependency distribution. Neither is a rewrite of your config, but both raised the minimum Node version, and v4's install model is the practical thing that breaks pipelines.

## When to Use / When Not

**Use when:**
- You are publishing a JavaScript/TypeScript **library** and want clean, small ESM/CJS/UMD output with precise control over externals.
- You need multiple output formats from one source in a single build.
- You want the best tree-shaking for hand-authored ES-module code.
- You are writing bundler plugins and want the API that Vite and unplugin build on.

**Avoid when:**
- You are building an **application** with a dev server, HMR, and assets — use Vite, which wraps Rollup for you.
- Raw build speed on a very large graph is the constraint — a Rust/Go bundler (esbuild, or Rolldown as it matures) will be faster.
- Your codebase is heavily CommonJS with dynamic requires — the interop tax may outweigh the tree-shaking benefit.

## Alternatives

- vitejs/vite — the application-layer answer; uses Rollup for production builds, so it is a superset for app authors rather than a competitor.
- evanw/esbuild — Go bundler, dramatically faster, less thorough tree-shaking and a smaller plugin surface; use when speed beats output precision.
- rolldown/rolldown — Rust reimplementation of the Rollup API from the Vite team; use when you want Rollup semantics with native speed and can tolerate pre-1.0 maturity.
- webpack/webpack — richer app/code-splitting/loader ecosystem; use for complex applications that predate or don't fit Vite.
- swc-project/swc / parcel-bundler/parcel — Rust and zero-config bundlers respectively; use when you want speed (swc) or no configuration (Parcel) over Rollup's control.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015 | Initial release by Rich Harris; introduced ESM-first bundling and the "tree-shaking" term[^1][^2]. |
| 1.0 | 2018-12 | First stable major; plugin API consolidation. |
| 2.0 | 2020-04 | Reworked plugin/output internals, faster watch. |
| 3.0 | 2022-11 | Raised minimum Node, config/CLI changes[^7]. |
| 4.0 | 2023-10 | SWC (Rust) parser replaces Acorn; native binaries via optional deps[^4]. |

## References

[^1]: Rollup project homepage and introduction. https://rollupjs.org/introduction/
[^2]: Rich Harris, "Tree-shaking versus dead code elimination." https://medium.com/@Rich_Harris/tree-shaking-versus-dead-code-elimination-d3765df85c80
[^3]: Vite documentation — "Using Plugins" (Vite's plugin API extends Rollup's). https://vite.dev/guide/api-plugin
[^4]: Rollup 4 release notes — migration to the SWC-based native parser. https://github.com/rollup/rollup/releases/tag/v4.0.0
[^5]: Rolldown — Rust bundler compatible with the Rollup API, from the Vite team. https://rolldown.rs/
[^6]: npm issue — optional dependencies and lockfiles can cause "Cannot find module @rollup/rollup-<platform>". https://github.com/rollup/rollup/issues/4699
[^7]: Rollup 3 release notes. https://github.com/rollup/rollup/releases/tag/v3.0.0

## Tags

javascript, bundler, es-modules, tree-shaking, build-tool, library-tooling, vite, plugin-api, frontend-tooling, esm
