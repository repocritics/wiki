# webpack/webpack

> The module bundler that defined the JavaScript build era — a graph compiler for anything you can `import`.

[GitHub repo](https://github.com/webpack/webpack) ·
[Official website](https://webpack.js.org) ·
[License: MIT](https://github.com/webpack/webpack/blob/main/LICENSE)

## Overview

Webpack is a bundler for JavaScript "and friends": it walks a dependency graph starting from one or more entry points, applies transformations, and emits a small number of bundled assets in place of many source modules[^1]. It was created by Tobias Koppers around 2012, and for most of the 2015–2021 period it was the default build tool for the JavaScript ecosystem — the thing `create-react-app`, Vue CLI, Angular CLI, and Next.js all shipped underneath. With ~66k stars and ~9.5k forks it remains one of the most-depended-on packages on npm, even as greenfield projects increasingly start elsewhere.

The defining idea is that **everything is a module**. Not just JS: through *loaders*, CSS, images, fonts, TypeScript, SVG, and arbitrary files become nodes in the same dependency graph, and through *plugins* nearly every step of the build is an interceptable hook. This generality is webpack's strength and its cost — it can bundle almost anything, but the configuration surface is large and the mental model (loaders, plugins, chunks, the module graph, the runtime) is nontrivial to hold in your head.

Webpack's central tension in 2026 is competitive: it is mature, battle-tested, and enormously configurable, but its Node-based, JavaScript-implemented pipeline is slower than the newer Go/Rust bundlers (esbuild, Rspack, Vite's Rolldown). It is no longer the default choice for new apps; it is the incumbent that large, long-lived codebases keep because migrating a heavily-customized webpack config is expensive.

## Getting Started

```bash
npm install --save-dev webpack webpack-cli
```

```js
// webpack.config.js
const path = require("path");

module.exports = {
  mode: "production",              // or "development"
  entry: "./src/index.js",
  output: {
    filename: "[name].[contenthash].js",
    path: path.resolve(__dirname, "dist"),
    clean: true,
  },
  module: {
    rules: [
      { test: /\.css$/i, use: ["style-loader", "css-loader"] },
      { test: /\.(png|svg|jpg)$/i, type: "asset/resource" },
    ],
  },
};
```

```bash
npx webpack --config webpack.config.js
```

For a dev server with hot reload, add `webpack-dev-server` and run `npx webpack serve`. Note that `webpack` and `webpack-cli` are separate packages — a missing `webpack-cli` is the single most common first-run error.

## Architecture / How It Works

Webpack's pipeline is a graph compiler driven almost entirely by a plugin/hook system (`tapable`)[^2]:

1. **Entry & resolution** — starting from `entry`, webpack resolves each `import`/`require` to a file using the `enhanced-resolve` algorithm (extensions, `resolve.alias`, `mainFields`, `node_modules` walking).
2. **Loaders** — each matched module is run through its loader chain, right-to-left. A loader is a function `string|Buffer -> string|Buffer`; `babel-loader`, `ts-loader`, `css-loader`, and `sass-loader` are the common ones. Loaders transform *individual files*.
3. **Module graph** — parsed modules (via `acorn`) yield further dependencies, building the full graph. Webpack understands ESM, CommonJS, and AMD, and can mix them.
4. **Chunking & optimization** — the graph is split into *chunks* (entry chunks, async chunks from dynamic `import()`, and shared chunks via `SplitChunksPlugin`). Tree-shaking removes unused ESM exports; `TerserPlugin` minifies.
5. **Plugins** — plugins tap into `compiler` and `compilation` hooks to influence *the whole build* (emit assets, inject the HTML, define globals, extract CSS). `HtmlWebpackPlugin`, `MiniCssExtractPlugin`, and `DefinePlugin` are near-universal.
6. **Runtime & emit** — webpack injects a small runtime (`__webpack_require__`) that implements module loading, chunk loading, and the module cache, then writes the output assets.

The mental model that trips people up: **loaders operate per-file and transform source; plugins operate per-build and hook the compiler.** They are not interchangeable. `mini-css-extract-plugin`, notably, is *both* — a loader half and a plugin half that must both be registered.

Two features carry disproportionate weight in real apps: **code splitting** (dynamic `import()` produces separately-loaded chunks) and **Module Federation** (added in webpack 5), which lets independently-built bundles share modules and consume each other at runtime — the foundation of most micro-frontend architectures[^3].

## Production Notes

**Build speed is the reason people leave.** Webpack's pipeline runs in Node and much of the hot path is JavaScript. Large apps see cold builds in the tens of seconds to minutes. Mitigations, roughly in order of impact: enable the persistent filesystem cache (`cache: { type: "filesystem" }`, standard since webpack 5 and a large win on warm builds), use `swc-loader` or `esbuild-loader` instead of `babel-loader`/`ts-loader`, use `thread-loader` for parallelism, and scope loader `test`/`include` narrowly so node_modules isn't reprocessed. When this stops being enough, teams migrate to Rspack (a Rust reimplementation with a near-compatible config) or Vite.

**Config sprawl.** A production webpack setup accumulates loaders, plugins, `optimization.splitChunks` tuning, `resolve.alias`, and environment branching. This config is a liability: it is verbose, order-sensitive, and the failure modes (a loader in the wrong order, a plugin registered but its loader missing) produce opaque errors. Tools like `webpack-merge` help split dev/prod configs but do not reduce the underlying surface.

**Source maps.** `devtool` choice is a real tradeoff between build speed, rebuild speed, and fidelity. `eval-cheap-module-source-map` for dev, `source-map` (separate files, slower) for production debugging. The wrong choice silently makes builds slow or stack traces useless.

**Bundle size discipline requires tooling.** Tree-shaking only works on ESM and is defeated by side-effectful modules unless `sideEffects: false` is declared in `package.json`. Use `webpack-bundle-analyzer` to find bloat; large accidental inclusions (moment locales, entire lodash, duplicate React copies) are the usual culprits.

**Upgrade pain: webpack 4 → 5.** The v5 migration (2020) removed automatic Node.js core polyfills (`crypto`, `stream`, `buffer` no longer auto-shimmed for the browser), changed the cache and asset-module handling, and broke many plugins until they published v5-compatible releases[^4]. Codebases pinned to webpack 4 with a wall of custom plugins were the slowest to move. Expect loader/plugin version alignment to dominate any major upgrade.

**You rarely configure webpack directly anymore.** In most apps it sits under Next.js, CRA, Vue CLI, or Angular's builder. Ejecting or overriding that config is where the real difficulty lives — and Next.js has itself moved its default bundler to Turbopack, meaning custom `webpack()` config there is increasingly a legacy path.

## When to Use / When Not

**Use when:**
- You maintain an existing app already on webpack — the config investment is real and Rspack/Vite migration is a project, not a flag.
- You need Module Federation for micro-frontends; webpack is the reference implementation.
- You need an unusual asset pipeline where the loader/plugin ecosystem's breadth matters more than raw speed.
- You want maximum control over chunking and output, and are willing to pay for it in config.

**Avoid when:**
- You're starting a new app and value dev-server startup and rebuild speed — Vite (Rollup/Rolldown + esbuild) is the current default there.
- You want fast builds with a webpack-shaped config — Rspack is a near drop-in with Rust performance.
- Your bundling needs are simple (a library, a small app) — esbuild or `tsup` are far less ceremony.
- The team lacks the appetite to own a large config; the abstraction leaks under pressure.

## Alternatives

- vitejs/vite — dev server on native ESM + esbuild, Rollup for production; the default for new frontend apps. Use instead when you want fast startup and minimal config.
- web-infra-dev/rspack — Rust reimplementation with a webpack-compatible config and plugin API. Use instead when you want webpack's model at much higher speed.
- evanw/esbuild — Go bundler, extremely fast, intentionally smaller feature set. Use instead when speed matters and you don't need webpack's plugin depth.
- rollup/rollup — ESM-first, clean output; the standard for bundling libraries. Use instead when shipping a package rather than an app.
- parcel-bundler/parcel — zero-config bundler. Use instead when you want sane defaults without writing config at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014-02 | First stable release; loader/plugin architecture established. |
| 2.0 | 2016-12 | Native ES module support, tree-shaking, config schema validation. |
| 3.0 | 2017-06 | Scope hoisting (`ModuleConcatenationPlugin`). |
| 4.0 | 2018-02 | Zero-config `mode`, `SplitChunksPlugin` replaces CommonsChunkPlugin, faster builds. |
| 5.0 | 2020-10 | Persistent caching, Module Federation, asset modules, removed Node polyfill auto-shims[^4]. |

Webpack 5 remains the current major line as of 2026; development has continued on it rather than a version 6, with effort focused on performance and the plugin ecosystem while much of the "next-generation bundler" energy moved to Rust/Go tools.

## References

[^1]: webpack README and documentation. https://webpack.js.org/concepts/
[^2]: webpack plugin & hook system (`tapable`). https://webpack.js.org/api/plugins/
[^3]: Module Federation concepts. https://webpack.js.org/concepts/module-federation/
[^4]: webpack, "To v5 from v4" migration guide. https://webpack.js.org/migrate/5/

## Tags

javascript, module-bundler, build-tool, compiler, code-splitting, module-federation, frontend, tree-shaking, loaders, plugins, web-performance
