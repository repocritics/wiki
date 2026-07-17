# unjs/unplugin

> Write a build-tool plugin once against a Rollup-style hook set, run it on Vite, Rollup, webpack, esbuild, Rspack, Rsbuild, Rolldown, Farm, and Bun.

[GitHub repo](https://github.com/unjs/unplugin) ·
[Official website](https://unplugin.unjs.io) ·
[License: MIT](https://github.com/unjs/unplugin/blob/main/LICENSE)

## Overview

unplugin is a thin abstraction layer that lets a single plugin definition target
many JavaScript build tools. You author one factory returning Rollup-flavored
hooks (`resolveId`, `load`, `transform`, `buildStart`, `buildEnd`,
`watchChange`), and unplugin produces the native plugin object each bundler
expects. It was created in 2021 by Anthony Fu and lives under the unjs
collective (the repo path `antfu/unplugin` now resolves to `unjs/unplugin`)[^1].

Its own star count (~3.6k) understates its reach: unplugin is the substrate
under much larger downstream projects — unplugin-icons, unplugin-vue-components,
and unplugin-auto-import each have more stars than unplugin itself, and Nuxt
modules lean on it heavily[^2]. When a tool advertises "works with Vite,
webpack, and Rspack," unplugin is usually why.

The defining tradeoff is lowest-common-denominator design. The unified hook set
is the intersection of what every supported bundler can express, so
bundler-specific power (Vite's dev-server hooks, webpack's compilation
internals, esbuild's `onResolve` namespaces) is only reachable through escape
hatches, not the portable core. unplugin trades ceiling for portability, and
whether that trade pays off depends entirely on how much bundler-specific
behavior your plugin needs.

## Getting Started

```bash
npm install unplugin
```

```ts
// my-plugin.ts
import { createUnplugin } from 'unplugin'

export const MyPlugin = createUnplugin<{ include?: RegExp }>((options, meta) => {
  // meta.framework === 'vite' | 'rollup' | 'webpack' | 'esbuild' | ...
  return {
    name: 'my-plugin',
    enforce: 'pre',
    transformInclude(id) {
      return (options.include ?? /\.ts$/).test(id)
    },
    transform(code, id) {
      return code.replace('__BUILD__', JSON.stringify(meta.framework))
    },
  }
})

// Consumers import the bundler-specific entry:
export const vite = MyPlugin.vite
export const webpack = MyPlugin.webpack
export const rollup = MyPlugin.rollup
export const esbuild = MyPlugin.esbuild
```

```ts
// vite.config.ts
import { MyPlugin } from './my-plugin'
export default { plugins: [MyPlugin.vite({ include: /\.vue$/ })] }
```

## Architecture / How It Works

The contract is Rollup's plugin interface, minus the parts other bundlers can't
honor. `createUnplugin` returns an object whose keys (`.vite`, `.rollup`,
`.webpack`, `.esbuild`, `.rspack`, `.rsbuild`, `.rolldown`, `.farm`) are factory
functions; each adapts your hook object to that tool's plugin shape.

Mapping difficulty varies sharply by target:

- **Rollup / Vite / Rolldown** — near-native. Vite's plugin API is a Rollup
  superset, so hooks pass through almost directly. This is the reference target,
  and the one where behavior most closely matches your mental model.
- **webpack / Rspack** — the hard case. webpack has no per-module `transform`
  hook; transforms are loaders matched by resource path. unplugin injects a
  virtual loader and uses `transformInclude(id)` to decide which modules the
  loader attaches to. This is why `transformInclude` exists at all: on Rollup you
  filter inside `transform`, but webpack needs the filter up front to register
  the loader.
- **esbuild** — the thinnest adapter. `resolveId` maps to `onResolve`,
  `load`/`transform` to `onLoad`. esbuild's plugin model lacks a full build
  lifecycle and ordering guarantees, so it is historically the least complete
  target.

A `transformInclude` filter is effectively mandatory for the loader-based
targets — without it, the transform runs on every module, which is both a
correctness and performance hazard.

The plugin context (`this`) is a normalized `UnpluginBuildContext` exposing a
subset of Rollup's context: `emitFile`, `addWatchFile`, `getWatchFiles`, plus
warning/error helpers. Not every Rollup context method exists on every backend,
so relying on advanced context APIs quietly reduces portability. For behavior
that genuinely needs a specific bundler, unplugin lets you attach raw
native-plugin config under per-target keys (e.g. a `vite` block for
`configureServer`/`handleHotUpdate`, a `webpack` block for compiler taps), which
runs only on that backend.

Recent versions adopt Rollup 4's object-form hook filters
(`transform: { filter, handler }`) so filtering can be pushed down into the
bundler where supported, rather than always running JS filter callbacks[^3].

## Production Notes

- **esbuild is the weak target.** If your plugin must run identically across all
  backends, test esbuild first — missing lifecycle hooks and limited transform
  chaining surface there before anywhere else. Treat "works on Vite" as no
  guarantee it works on esbuild.
- **`transformInclude` is not optional in practice.** On webpack/Rspack/esbuild,
  omitting it means your transform touches every module. Scope it tightly by
  extension or path; this is the single most common performance footgun.
- **Vite dev prebundling skips your plugin.** Vite pre-bundles dependencies with
  esbuild during dev; transforms written for the Rollup-style pipeline do not run
  on prebundled `node_modules` deps. Plugins that must see dependency code need a
  separate esbuild-plugin path via the `vite` escape hatch or `optimizeDeps`
  config.
- **Sourcemap fidelity differs by backend.** Returning `{ code, map }` from
  `transform` behaves cleanly on Rollup/Vite; the loader-wrapped webpack path can
  drop or mangle maps if the map isn't well-formed. Verify maps on webpack, not
  just Vite.
- **`enforce` ordering is approximate.** `pre`/`post` map to each bundler's own
  ordering model, which is not identical. esbuild in particular has weaker
  ordering guarantees than Vite/webpack.
- **Version jumps carry migration cost.** v2 (2024-12) and v3 (2026-01) were
  major-version bumps that raised minimum tooling/runtime baselines and
  reorganized backend support; pin unplugin and its consuming plugins together
  when upgrading, since downstream plugins track the major line[^4].

## When to Use / When Not

**Use when:**
- You are publishing a build-tool plugin and want one codebase to serve the whole
  ecosystem instead of maintaining parallel Vite/webpack/Rollup packages.
- Your plugin's work is expressible as resolve/load/transform — code
  transformation, virtual modules, macros, auto-imports, icon compilation.
- You want your users to adopt you regardless of which bundler they run.

**Avoid when:**
- You only ever target one bundler — write that bundler's native plugin and get
  its full API with no abstraction tax.
- Your plugin depends on bundler-internal features (webpack compilation graph,
  Vite HMR boundaries, esbuild namespaces) as its core, not as an add-on.
- You need guaranteed identical behavior across backends for something subtle;
  the intersection semantics leak, especially on esbuild.

## Alternatives

- rollup/rollup — write a native Rollup plugin directly when Rollup/Vite are your
  only real targets and you want the complete, unabstracted hook set.
- vitejs/vite — author a native Vite plugin when you need dev-server, HMR, or
  config hooks that live outside unplugin's portable core.
- evanw/esbuild — use esbuild's own plugin API when esbuild is the sole target
  and you need its resolve namespaces and speed without an adapter in the way.
- web-infra-dev/rspack — target Rspack/webpack loaders directly when the webpack
  ecosystem is where you live and cross-bundler reach is irrelevant.
- rolldown/rolldown — for Rollup-compatible plugins aimed at the next-gen Rust
  bundler specifically; unplugin also emits Rolldown plugins if you want both.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021-08-25 | First tagged release; Vite/Rollup/webpack/esbuild targets[^1]. |
| 1.0.0 | 2022-11-14 | API stabilized. |
| 2.0.0 | 2024-12-05 | Major bump; raised baselines, reorganized backend support[^4]. |
| 3.0.0 | 2026-01-22 | Second major; expanded backends (Rspack/Rsbuild/Rolldown/Farm/Bun), hook-filter adoption[^3]. |
| 3.3.0 | 2026-06-29 | Latest at time of writing; active regular cadence. |

## References

[^1]: unjs/unplugin repository and README — "Unified plugin system for build tools." https://github.com/unjs/unplugin
[^2]: unplugin org ecosystem (unplugin-icons, unplugin-vue-components, unplugin-auto-import). https://github.com/unplugin
[^3]: unplugin documentation, hook filters and supported bundlers. https://unplugin.unjs.io
[^4]: unplugin releases (v2.0.0, v3.0.0 major-version notes). https://github.com/unjs/unplugin/releases

## Tags

typescript, build-tools, bundler-plugin, plugin-abstraction, vite, rollup, webpack, esbuild, rspack, rolldown, unjs
