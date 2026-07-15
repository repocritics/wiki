# web-infra-dev/rspack

> A Rust-based bundler that reimplements webpack's configuration and plugin API, aiming to be a mostly-drop-in replacement at higher speed.

[GitHub repo](https://github.com/web-infra-dev/rspack) ·
[Official website](https://rspack.rs) ·
[License: MIT](https://github.com/web-infra-dev/rspack/blob/main/LICENSE)

## Overview

Rspack is a bundler written in Rust and developed by ByteDance's web infrastructure team[^1]. Its defining bet is compatibility rather than reinvention: it deliberately mirrors webpack's `Configuration` shape, its loader interface, and much of its plugin hook system (via `tapable`-style hooks), so that large existing webpack projects can migrate by swapping the bundler and keeping most of their config[^2]. This distinguishes it from esbuild and Rolldown, which offer their own (smaller) APIs, and from Vite, which pairs esbuild for dev with Rollup for production.

The tradeoff behind that bet is real. webpack's API is enormous and was never designed as a spec, so "webpack-compatible" is a moving target: Rspack implements the commonly used surface well but does not cover every plugin, every hook, or every edge of webpack's behavior. Teams migrating hit a long tail of plugins that assume internals Rspack does not expose. In exchange, builds and hot-module-replacement are substantially faster because the graph, parsing (via SWC), and code generation run in native Rust threads instead of single-threaded JavaScript[^3].

Rspack is the core of "Rstack", a family of tools built on top of it: Rsbuild (a higher-level, config-light build tool), Rslib (library builds), Rspress (docs/SSG), Rsdoctor (build analysis), Rstest, and Rslint[^4]. Most application developers are better served by Rsbuild than by hand-writing Rspack config; Rspack itself is the low-level engine.

## Getting Started

```bash
npm install -D @rspack/core @rspack/cli
```

```js
// rspack.config.js — intentionally webpack-shaped
const rspack = require("@rspack/core");

module.exports = {
  entry: "./src/index.js",
  module: {
    rules: [
      { test: /\.css$/, type: "css" },            // native CSS handling
      {
        test: /\.tsx?$/,
        use: {
          loader: "builtin:swc-loader",            // in-process SWC, no babel
          options: { jsc: { parser: { syntax: "typescript", tsx: true } } },
        },
        type: "javascript/auto",
      },
    ],
  },
  plugins: [new rspack.HtmlRspackPlugin({ template: "./index.html" })],
};
```

For most apps, prefer the higher-level entry point instead of writing the above by hand:

```bash
npm create rsbuild@latest
```

## Architecture / How It Works

Rspack's core is a Rust workspace exposed to Node.js through NAPI-RS bindings[^5]. JavaScript configures and drives the build; the module graph construction, dependency resolution, chunk splitting, and code generation happen in Rust. Parsing, transformation, and minification are delegated to SWC rather than Babel/Terser, which removes the largest JS-side cost in a typical webpack pipeline.

Compatibility is achieved by re-implementing webpack concepts natively:

- **Loaders** run as JavaScript on the Node side (so existing loaders like `sass-loader` work), but performance-critical ones ship as `builtin:` loaders that execute inside Rust — `builtin:swc-loader` replaces `babel-loader`, and much CSS handling is native rather than requiring the `css-loader`/`style-loader` chain.
- **Plugins** tap lifecycle hooks that intentionally echo webpack's `compiler`/`compilation` hooks. The bridge marshals hook calls across the Rust/JS boundary, which is why some hooks are supported and some are not — each must be explicitly wired.
- **Module Federation** is treated as a first-class concern, reflecting ByteDance's micro-frontend usage and a collaboration with Module Federation's author[^1].

The central engineering tension is the Rust↔JS boundary. Every plugin hook or JS loader invocation crosses NAPI, and crossing it has cost; Rspack's speed comes largely from doing as much as possible on the Rust side and minimizing round-trips. This also shapes what "compatible" can mean — a webpack plugin that leans on synchronous access to deep internal data structures may be impractical to support even when its hook exists.

## Production Notes

- **"Drop-in" is aspirational, not literal.** Simple projects migrate cleanly; complex webpack configs with many bespoke plugins do not. Budget migration time for auditing every plugin and loader against Rspack's compatibility list, and expect to replace a few.
- **Prefer `builtin:` loaders.** Using `babel-loader` or the classic CSS loader chain works but throws away much of the speed advantage, because work moves back onto the JS thread. The performance story assumes `builtin:swc-loader` and native CSS/asset handling.
- **SWC is not Babel.** Because transformation is SWC-based, custom Babel plugins and certain macro-style transforms have no direct equivalent. Code depending on specific Babel plugin behavior needs rework or a JS `babel-loader` fallback.
- **Version cadence is aggressive.** v0.x → v1.0 (2024) → v2.0 (2026) each carried breaking changes and config migrations; the ecosystem of Rspack-specific plugins must track these. Pin versions and read migration guides before major upgrades.
- **Ecosystem is younger than webpack's.** The webpack plugin catalog is vast and battle-tested over a decade; Rspack's native plugin set and community coverage, while growing quickly, is smaller. Niche needs may lack a maintained option.
- **Rsbuild absorbs most footguns.** If you do not have a strong reason to control raw Rspack config, Rsbuild provides sane defaults and hides most of the compatibility sharp edges.

## When to Use / When Not

**Use when:**
- You have an existing webpack project whose build times hurt and whose plugin/loader set is mainstream.
- You use Module Federation and want first-class, fast support.
- You want webpack's mental model (loaders, plugins, chunk config) but native-speed builds.
- You are starting fresh and want the Rsbuild-on-Rspack experience.

**Avoid when:**
- Your build depends on many niche webpack plugins or custom Babel transforms with no Rspack equivalent.
- You want the smallest, simplest config and don't need webpack compatibility — esbuild or Vite may fit better.
- You are shipping a library and want a Rollup-style output — reach for Rslib, tsup, or Rollup directly.
- You need a decade-stable API surface; Rspack still ships breaking major versions.

## Alternatives

- webpack/webpack — use instead when you depend on plugins Rspack doesn't yet support, or need the most mature ecosystem and can tolerate slower builds.
- vitejs/vite — use instead for greenfield apps wanting fast dev via native ESM and a Rollup production build, without webpack compatibility.
- evanw/esbuild — use instead when you want the fastest raw bundling with a small API and don't need webpack's plugin model.
- rolldown/rolldown — use instead when you want a Rust, Rollup-compatible bundler (and it is Vite's future production bundler).
- privatenumber/tsup — use instead for simple library builds where a thin esbuild wrapper is enough.

## History

| Version | Date | Notes |
|---------|------|-------|
| Repo created | 2022-04-01 | Development begins inside ByteDance web-infra. |
| 0.1.0 | 2023-03-10 | First public release; webpack-compatibility focus announced[^1]. |
| 1.0.0 | 2024-08-28 | First stable major; API/plugin surface declared production-ready[^6]. |
| 2.0.0 | 2026-04-22 | Second major; breaking changes and continued Rstack consolidation. |

## References

[^1]: Rspack announcement, "Announcing Rspack 0.1". https://rspack.rs/blog/announcing-0-1
[^2]: Rspack docs, "Introduction". https://rspack.rs/guide/start/introduction
[^3]: Rspack README, "Fast Rust-based bundler for the web with a modernized webpack API". https://github.com/web-infra-dev/rspack
[^4]: Rstack toolchain overview (README "Rstack" section). https://github.com/web-infra-dev/rspack#-rstack
[^5]: NAPI-RS — Node.js addons in Rust, used for Rspack's node bindings. https://napi.rs
[^6]: Rspack blog, "Announcing Rspack 1.0". https://rspack.rs/blog/announcing-1-0

## Tags

rust, javascript, bundler, build-tool, webpack-compatible, module-federation, swc, frontend-tooling, web-performance, napi, module-bundler
