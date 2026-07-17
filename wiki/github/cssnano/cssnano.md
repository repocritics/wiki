# cssnano/cssnano

> A modular CSS minifier built as a PostCSS plugin — the default minification
> step of the webpack era, now competing with far faster Rust tools.

[GitHub repo](https://github.com/cssnano/cssnano) · [Official website](https://cssnano.github.io/cssnano/) · [License: MIT](https://github.com/cssnano/cssnano/blob/master/LICENSE-MIT)

## Overview

cssnano is a CSS minifier assembled from ~30 individual PostCSS plugins, each
responsible for one transform: stripping comments, shortening colors, merging
longhand properties, deduplicating rules, folding `calc()`, compressing SVG
data URIs via SVGO. Started by Ben Briggs in April 2015[^1], it rode the
PostCSS wave to become the implicit standard: `css-minimizer-webpack-plugin`
uses it by default, so most webpack projects run cssnano without naming it.

The defining design choice is the **preset system** (since v4, 2018[^2]):
`default` applies only transforms safe for any stylesheet, `lite` does little
more than whitespace and comments, and `advanced` adds behavior-changing
transforms — z-index rebasing, identifier renaming, unused at-rule removal —
that are only safe if you control every stylesheet and script on the page —
an honest admission that minification beyond whitespace is never fully
semantics-preserving.

The defining tension in 2026 is speed. cssnano is single-threaded JavaScript
walking a PostCSS AST; Lightning CSS (Rust) and esbuild (Go) are dramatically
faster and have displaced it as the default in Parcel 2, Vite, and Tailwind
v4. At ~5.0k stars with steady, low-volume maintenance, cssnano is a mature
incumbent holding the webpack/PostCSS niche rather than a growing project.
(GitHub lists the repo language as "CSS" — a test-fixture artifact; the
source is JavaScript.)

## Getting Started

```bash
npm install cssnano postcss postcss-cli --save-dev
npx postcss input.css > output.min.css   # after configuring:
```

```js
// postcss.config.js
module.exports = {
  plugins: [
    require("cssnano")({
      preset: "default", // or "advanced" / "lite" (separate packages)
    }),
  ],
};
```

With webpack, use `css-minimizer-webpack-plugin`, which wraps cssnano.

## Architecture / How It Works

The `cssnano` package itself is a thin orchestrator: it resolves a preset —
an ordered list of PostCSS plugins with options — and hands it to the PostCSS
runner. All real work happens in the plugins, maintained in one monorepo:

- **Structure**: `postcss-discard-comments`, `postcss-discard-duplicates`,
  `postcss-merge-rules`, `postcss-merge-longhand` (four `margin-*` → one).
- **Values**: `postcss-colormin` (`#ff0000` → `red`), `postcss-calc`
  (constant folding), `postcss-convert-values`, `postcss-minify-gradients`.
- **Selectors**: `postcss-minify-selectors`, `postcss-unique-selectors`.
- **Embedded assets**: `postcss-svgo` runs SVGO over inline SVG data URIs.
- **Advanced-only**: `postcss-zindex` (rebases z-index scales),
  `postcss-reduce-idents` (renames `@keyframes`/counter identifiers),
  `postcss-discard-unused`, `postcss-merge-idents`, plus autoprefixer to
  strip prefixes your Browserslist targets no longer need.

Some transforms (`postcss-colormin`, `postcss-reduce-initial`) are
Browserslist-aware. Plugin order within a preset is fixed and load-bearing —
longhand merging enables rule merging enables duplicate discarding — hence
curated presets over user-composed plugin lists. Because everything is a
PostCSS AST pass, cssnano composes with the rest of a PostCSS chain
(autoprefixer, preset-env, Tailwind v3) in a single parse — the main
structural argument over standalone minifiers.

## Production Notes

**The advanced preset breaks real sites.** `postcss-zindex` rebases z-index
values assuming your stylesheet is the only source of stacking order; any
third-party widget, JS-set `zIndex`, or second stylesheet breaks that
assumption silently. `postcss-reduce-idents` renames `@keyframes`, breaking
animations referenced from JavaScript. `postcss-discard-unused` cannot see
dynamically injected HTML. Treat `advanced` as opt-in per-transform: disable
`zindex`, `reduceIdents`, and `discardUnused` unless verified.

**The default preset is safe by intent, not by proof.** `postcss-merge-longhand`
and `postcss-calc` have historically produced regressions around CSS custom
properties and mixed-unit expressions. When chasing a "minified CSS renders
differently" bug, bisect by disabling preset features before blaming your CSS.

**Performance is the reason teams leave.** On framework-scale CSS, cssnano is
commonly the slowest step in the pipeline; Lightning CSS and esbuild are
roughly an order of magnitude faster[^3]. But if you already run PostCSS,
cssnano's marginal cost is one extra pass over an AST you have parsed anyway
— harder to beat than raw benchmarks suggest.

**Dependency churn.** SVGO is a heavyweight transitive dependency and a
recurring source of audit warnings and breaking-range bumps. The monorepo
publishes ~30 packages in lockstep; version mismatches between `cssnano` and
`cssnano-preset-*` after partial upgrades are a known footgun.

**Upgrade pains.** v4 (2018) replaced ad-hoc options with the preset system —
old configs did not carry over[^2]. v5 (2021) required the PostCSS 8 plugin
API, forcing simultaneous upgrades across the whole PostCSS chain. v6 and v7
were mostly Node floor raises. v8 (2026-05) moved `css-declaration-sorter`
from the default preset to advanced and dropped Node 20 — default-preset
output ordering changed, which breaks snapshot tests and size baselines[^4].

## When to Use / When Not

**Use when:**
- You already run a PostCSS pipeline and want minification in the same pass.
- You use webpack — `css-minimizer-webpack-plugin` gives you cssnano free.
- You need per-transform control — presets expose options for every plugin.
- You need Browserslist-aware output minification.

**Avoid when:**
- CSS minification time is a measurable share of your build — Lightning CSS
  or esbuild do the same job far faster.
- You are on Vite, Parcel 2, or Tailwind v4 — the built-in minifiers already
  cover the default preset's ground.
- You want the advanced preset on a page you don't fully control — the
  unsafe transforms are the main source of cssnano-related incidents.

## Alternatives

- parcel-bundler/lightningcss — Rust minifier + transpiler; use when build
  speed matters or you want minification and syntax lowering in one tool.
- evanw/esbuild — built-in CSS minification (Vite's default); use when "good
  enough" minification without an extra dependency wins.
- clean-css/clean-css — standalone JS minifier; use for a lighter install
  outside a PostCSS chain.
- css/csso — structural optimizer with aggressive rule restructuring; use
  when maximum byte savings outweigh transform risk.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015-04 | Initial release by Ben Briggs[^1]. |
| 4.0 | 2018-07 | Preset system (default/advanced), monorepo restructure[^2]. |
| 5.0 | 2021-04 | PostCSS 8 plugin API required. |
| 6.0 | 2023-03 | Node 14+ floor, dependency modernization. |
| 7.0 | 2024-04 | Node 18+ floor. |
| 8.0 | 2026-05 | Declaration sorting moved to advanced preset; Node 20 dropped[^4]. |

## References

[^1]: Repo created 2015-04-14; `cssnano@1.0.0` published to npm the same day. https://github.com/cssnano/cssnano
[^2]: cssnano docs, presets and configuration. https://cssnano.github.io/cssnano/docs/getting-started
[^3]: Lightning CSS benchmarks (self-published; directionally consistent with independent reports). https://lightningcss.dev/
[^4]: cssnano@8.0.0 release notes — 2026-05-06. https://github.com/cssnano/cssnano/releases/tag/cssnano%408.0.0

## Tags

css, minification, postcss, javascript, nodejs, build-tools, frontend, optimization, monorepo, webpack
