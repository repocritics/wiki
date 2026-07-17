# rollup/plugins

> The official Rollup plugin monorepo — the `@rollup/plugin-*` packages that make Rollup usable against the real (CommonJS-infested) npm ecosystem.

[GitHub repo](https://github.com/rollup/plugins) ·
[Rollup plugin docs](https://rollupjs.org/plugin-development/) ·
[License: MIT](https://github.com/rollup/plugins/blob/master/LICENSE)

## Overview

`rollup/plugins` is the monorepo where the Rollup organization maintains its official plugins under the `@rollup/plugin-*` npm scope. It was created in August 2019 to consolidate the sprawl of individually-maintained `rollup-plugin-*` community packages that Rollup could not function without — most importantly `node-resolve` (Node module resolution) and `commonjs` (CJS-to-ESM conversion)[^1]. Rollup's core is deliberately minimal: it bundles standard ES modules and nothing else. Everything a real project needs — resolving from `node_modules`, importing JSON, transpiling TypeScript, minifying — lives here.

The repo's GitHub stats undersell it. 3,757 stars is modest, but `@rollup/plugin-commonjs` and `@rollup/plugin-node-resolve` each see ~16M npm downloads per week, and `@rollup/pluginutils` (the shared helper library, home of `createFilter` and `dataToEsm`) sees ~53M[^2] — largely because Vite uses `@rollup/plugin-commonjs` internally for production builds and virtually every Vite/Rollup plugin depends on `pluginutils`. This is load-bearing infrastructure for the JavaScript toolchain, maintained by a small volunteer team.

The defining tension: the hardest plugins here solve a problem that cannot be solved cleanly. Statically converting arbitrary CommonJS (dynamic `require`, mutated `module.exports`, circular deps) into ES modules is heuristic by nature, so `@rollup/plugin-commonjs` carries a long tail of interop options and edge-case bugs that no rewrite has fully eliminated.

## Getting Started

```bash
npm install --save-dev rollup @rollup/plugin-node-resolve @rollup/plugin-commonjs @rollup/plugin-json
```

```js
// rollup.config.mjs
import { nodeResolve } from '@rollup/plugin-node-resolve';
import commonjs from '@rollup/plugin-commonjs';
import json from '@rollup/plugin-json';

export default {
  input: 'src/index.js',
  output: { file: 'dist/bundle.js', format: 'es' },
  plugins: [
    nodeResolve(),   // resolve bare imports from node_modules
    commonjs(),      // AFTER nodeResolve — convert CJS deps to ESM
    json(),
  ],
};
```

Each plugin is published and versioned independently (`@rollup/plugin-commonjs`, `@rollup/plugin-typescript`, ...); install only what you use.

## Architecture / How It Works

The repo is a pnpm monorepo with ~30 packages under `packages/`, each independently versioned, tagged (`name-vX.Y.Z`), and published by a shared release script[^1]. There is no lockstep versioning — a fix to `json` does not bump `commonjs` — which keeps churn low but makes "which versions go together" a peer-dependency question rather than a single number.

All plugins are implementations of Rollup's hook-based plugin API[^3]: `resolveId` (where does this import live), `load` (give me its source), and `transform` (rewrite it) are the main hooks used here. Notable internals:

- **`node-resolve`** reimplements Node's module resolution in JS: `package.json` `main`/`module`/`browser` fields, the `exports` conditional-exports map, symlink handling, and `dedupe`. It is the reference for how bundlers interpret npm packaging metadata.
- **`commonjs`** parses each CJS module's AST, detects `require`/`module.exports` patterns, and emits an ESM wrapper plus a proxy module for interop. Named-export detection is static analysis over dynamic behavior — hence options like `requireReturnsDefault`, `defaultIsModuleExports`, `transformMixedEsModules`, and `strictRequires` that shift the heuristics per-dependency.
- **`pluginutils`** is the de facto SDK for third-party plugin authors: `createFilter` (include/exclude glob matching), `dataToEsm` (serialize data as a tree-shakeable module), `attachScopes`.
- Because these plugins stick to the standard build hooks, most of them also work unmodified in Vite, which implements the Rollup plugin interface.

Coupling story: every package declares a `rollup` peer dependency range. When Rollup ships a major (3.0 in October 2022, 4.0 in October 2023[^4]), the monorepo does a coordinated wave of major bumps, and downstream users get a matrix-compatibility upgrade chore.

## Production Notes

- **Plugin order matters and is not validated.** `commonjs()` must come after `nodeResolve()`; `babel`/`typescript` transforms generally run after both. Wrong order produces confusing resolution errors, not a clear diagnostic.
- **`commonjs` is where builds go to die.** Mixed ESM/CJS packages, dependencies that mutate `exports` at runtime, and dynamic `require(variable)` calls all break static conversion. Expect to reach for `transformMixedEsModules`, `dynamicRequireTargets`, or per-package `requireReturnsDefault` tuning; some dependencies simply have to be marked `external`. Interop bugs between the `default` export and namespace imports are the most common issue class[^5].
- **`@rollup/plugin-typescript` transpiles and type-checks with `tsc`, which is slow**, and it constrains `tsconfig` (`declarationDir` must sit inside Rollup's output directory; some options are overridden). Many production setups swap in esbuild- or SWC-based transpilation (`@rollup/plugin-swc` exists in this repo) and run `tsc --noEmit` separately.
- **Version-matrix pain on Rollup majors.** Old plugin majors silently misbehave or warn against new Rollup cores; upgrading Rollup means auditing every `@rollup/plugin-*` peer range in the lockfile.
- **`@rollup/plugin-babel` requires an explicit `babelHelpers` option** (`'bundled'` for apps, `'runtime'` for libraries); this is a deliberate breaking choice from the `rollup-plugin-babel` migration and a recurring migration stumble.
- **Some packages are legacy, not recommendations**: `buble`, `beep`, `auto-install`, and `legacy` date from an earlier era and see minimal attention. Presence in the monorepo does not mean "current best practice."
- **Maintenance cadence is slow but real.** Last push May 2026 with 39 open issues; releases arrive in bursts from a handful of maintainers. Fine for stable infrastructure, frustrating if you are waiting on an edge-case `commonjs` fix — check the issue tracker before betting a deadline on one.

## When to Use / When Not

**Use when:**
- You bundle anything with Rollup that touches `node_modules` — `node-resolve` + `commonjs` are effectively mandatory, and the `@rollup/` scope is the maintained lineage.
- You are writing a Rollup or Vite plugin — build on `@rollup/pluginutils` rather than reimplementing filtering.
- You publish a JS/TS library and want tree-shakeable ESM output with explicit control over externals and output formats.

**Avoid when:**
- You are building an application, not a library — Vite (dev server + sensible defaults) wraps this stack for you.
- Build speed dominates: prefer esbuild/SWC-based transpile plugins over `babel`/`typescript` from this repo, or a different bundler entirely.
- You are betting on Rolldown (the Rust-based Rollup successor), which ships built-in equivalents of `node-resolve`, `commonjs`, and `json` — this repo's core plugins become redundant there.

## Alternatives

- rolldown/rolldown — use instead when you want Rollup semantics at Rust speed; resolution/CJS/JSON handling is built in rather than plugged in.
- egoist/rollup-plugin-esbuild — use instead of `@rollup/plugin-babel`/`typescript` when transpile speed matters and you type-check separately.
- evanw/esbuild — use instead of the whole Rollup+plugins stack when you want an all-in-one fast bundler and accept less output control.
- vitejs/vite — use instead for application development; it embeds this plugin ecosystem behind an opinionated dev/build pipeline.
- unjs/unplugin — use instead of `pluginutils` alone when authoring plugins that must run across Rollup, Vite, webpack, and esbuild.

## History

| Version | Date | Notes |
|---------|------|-------|
| Repo created | 2019-08 | Monorepo established; official plugins consolidated under the `@rollup/` npm scope[^1]. |
| First scoped releases | 2019–2020 | `rollup-plugin-*` packages migrated with major bumps and breaking option changes (e.g. `babel`'s required `babelHelpers`). |
| `exports` support | 2020 | `node-resolve` adds Node conditional-exports (`exports` field) resolution. |
| Rollup 3 wave | 2022-10 | Coordinated major bumps for Rollup 3 peer compatibility[^4]. |
| `terser` adopted | 2022-11 | `@rollup/plugin-terser` created in-repo, superseding the unmaintained third-party `rollup-plugin-terser`. |
| Rollup 4 wave | 2023-10 | Peer-range updates for Rollup 4 (SWC-based parser)[^4]. |
| Maintenance mode | 2024–2026 | Steady low-volume fixes; last push 2026-05, 39 open issues. |

## References

[^1]: rollup/plugins README — scope, plugin list, monorepo/publish workflow. https://github.com/rollup/plugins#readme
[^2]: npm downloads API, week 2026-07-10 to 2026-07-16: `@rollup/plugin-node-resolve` 16.1M, `@rollup/plugin-commonjs` 16.6M, `@rollup/pluginutils` 53.2M. https://api.npmjs.org/downloads/point/last-week/@rollup/pluginutils
[^3]: Rollup plugin development documentation (hook API). https://rollupjs.org/plugin-development/
[^4]: Rollup releases — v3.0.0 (2022-10), v4.0.0 (2023-10). https://github.com/rollup/rollup/releases
[^5]: `@rollup/plugin-commonjs` README — interop options and limitations. https://github.com/rollup/plugins/tree/master/packages/commonjs

## Tags

javascript, rollup, bundler, build-tools, plugins, monorepo, commonjs, esm, module-resolution, npm-ecosystem
