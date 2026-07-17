# postcss/autoprefixer

> The PostCSS plugin that adds and removes CSS vendor prefixes from Can I Use data — the tool that made hand-written `-webkit-` a code smell.

[GitHub repo](https://github.com/postcss/autoprefixer) ·
[Interactive demo](https://autoprefixer.github.io/) ·
[License: MIT](https://github.com/postcss/autoprefixer/blob/main/LICENSE)

## Overview

Autoprefixer is a PostCSS plugin that parses stylesheets and inserts vendor
prefixes (`-webkit-`, `-moz-`, `-ms-`, `-o-`) for exactly the browsers you
target, using support data from Can I Use[^1]. You write standard CSS; the
build step adds whatever prefixes your Browserslist config demands and strips
outdated ones. Created by Andrey Sitnik in 2013, it is historically important
beyond its own function: PostCSS itself was written by the same author to give
Autoprefixer a proper CSS parser, and Browserslist was extracted from
Autoprefixer's browser-query logic before becoming the shared target-browser
config for Babel, ESLint plugins, and Stylelint[^2][^3].

The defining tension in 2026 is that Autoprefixer is both ubiquitous and
decreasingly necessary. Evergreen browsers ship most features unprefixed, so
for a modern Browserslist target the plugin often emits only a handful of
`-webkit-` lines (and its most valuable act is *removing* stale prefixes from
legacy code). It remains in nearly every CSS pipeline because it is the safe
default with near-zero configuration — but Rust-based tools now do the same
transform an order of magnitude faster. The repo reflects mature-project
economics: ~22.3k stars, 38 open issues, pushes as recent as this week, and a
release cadence that mostly tracks caniuse data rather than new features.

## Getting Started

```bash
npm install --save-dev postcss autoprefixer
```

```js
// postcss.config.js — picked up by webpack (postcss-loader), Vite, Next.js, etc.
module.exports = {
  plugins: [require('autoprefixer')]
}
```

Target browsers belong in a shared config, not plugin options[^1]:

```jsonc
// package.json
{ "browserslist": ["> 0.5%", "last 2 versions", "not dead"] }
```

## Architecture / How It Works

Autoprefixer is a pure AST transform on top of PostCSS. PostCSS parses CSS into
a node tree; Autoprefixer walks declarations, at-rules, and selectors, and
inserts or deletes prefixed clones. Three data layers drive the decisions:

1. **caniuse-lite** — compressed Can I Use support tables, consumed at build
   time of the package into a prefix database (`data/prefixes.js`).
2. **Browserslist** — resolves your query (`"> 0.5%"`, `.browserslistrc`) into
   a concrete browser list; only prefixes required by that list are applied[^3].
3. **Hacks** — a directory of per-feature classes for everything that is not a
   mechanical `property → -webkit-property` rename: the 2009/2012/final flexbox
   syntaxes, gradient direction rewrites, `::placeholder` selector variants,
   and the IE 10/11 Grid translation.

Two behaviors surprise newcomers. First, removal is on by default
(`remove: true`): Autoprefixer deletes prefixes it considers outdated for your
targets, so hand-written hacks vanish unless placed *after* the unprefixed
property or guarded with `/* autoprefixer: off */` control comments. Second, it
only prefixes from the unprefixed form — legacy `-webkit-`-only code gains
nothing without a preceding unprefix pass. It deliberately adds no polyfills;
it is prefixes only, which is why postcss-preset-env wraps it rather than the
reverse[^1].

The Grid story is the largest hack: `grid: "autoplace"` translates modern Grid
syntax into IE 10/11 `-ms-` syntax, including simulated autoplacement via
generated `-ms-grid-row/column` coordinates. It is off by default because the
translation is static — auto-fit/auto-fill, implicit tracks, and
`::before`/`::after` inside autoplaced grids all break silently[^4].

## Production Notes

- **"caniuse-lite is outdated" warnings.** The prefix data is baked into the
  installed package via caniuse-lite; lockfiles pin it for months. Run
  `npx update-browserslist-db@latest` periodically, or output drifts from
  reality (both missing and superfluous prefixes)[^5].
- **Your Browserslist config is the real behavior switch.** No config means
  Browserslist defaults; editing targets for one tool (Babel) silently changes
  CSS output. Diff built CSS when touching browserslist.
- **PostCSS 8 boundary.** Autoprefixer 10 (2020) requires `postcss` as a peer
  dependency and the PostCSS 8 visitor API. Old toolchains (webpack 4-era
  postcss-loader, some legacy CLIs) need autoprefixer 9.x, which no longer
  receives data updates.
- **Build cost.** It is JavaScript doing full CSS parse + walk per file. On
  large stylesheets in hot rebuild paths this is measurable; Lightning CSS
  performs the equivalent transform in Rust dramatically faster, and Tailwind
  CSS v4 dropped the PostCSS/Autoprefixer pipeline for exactly this reason.
- **IE Grid mode is not fire-and-forget.** Teams enabling `grid: "autoplace"`
  for legacy IE support must visually test every grid; the documented
  limitations list is long and violations fail without warnings[^4]. The
  maintainers also warn against editor plugins: prefixes belong in build
  output, not source.

## When to Use / When Not

**Use when:**
- You run any PostCSS-based pipeline (webpack, Vite, Next.js defaults) and need
  correct prefixes without thinking about them.
- You maintain legacy CSS full of stale hand-written prefixes — the removal
  pass alone pays for itself.
- You must support old browser matrices (enterprise IE 11, old Android
  WebViews) where prefix coverage still matters.

**Avoid when:**
- Your target is evergreen-only and your bundler already handles the few
  remaining `-webkit-` cases (esbuild, Lightning CSS) — an extra JS pass buys
  nothing.
- Build performance is a bottleneck: Lightning CSS subsumes this transform at
  Rust speed.
- You expect polyfills for unsupported features — Autoprefixer adds prefixes
  only, by design.

## Alternatives

- parcel-bundler/lightningcss — Rust prefixer + transpiler + minifier in one
  pass; use it when build speed matters or you're on Parcel / Tailwind v4.
- csstools/postcss-plugins — postcss-preset-env bundles Autoprefixer plus
  future-CSS transpilation; use it when you want new syntax, not just prefixes.
- evanw/esbuild — built-in (narrower) CSS prefixing tied to `target`; enough
  when esbuild already owns your CSS and needs are basic.
- thysultan/stylis — runtime prefixer used by styled-components/Emotion; use it
  when CSS is generated in the browser and there is no build step.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013 | Released by Andrey Sitnik, originally on the Rework parser[^2]. |
| 1.0 | 2014 | Rewritten on PostCSS, created by the same author for this purpose[^2]. |
| 9.4 | 2018-12 | IE Grid autoplacement translation (`grid: "autoplace"`)[^4]. |
| 10.0 | 2020-09 | PostCSS 8 plugin API; `postcss` moved to peer dependency[^6]. |
| 10.4 | 2021-11 | Current major.minor line; subsequent patch releases mostly track caniuse data. |

## References

[^1]: Autoprefixer README. https://github.com/postcss/autoprefixer#readme
[^2]: PostCSS — created to power Autoprefixer; project history in the PostCSS repo. https://github.com/postcss/postcss
[^3]: Browserslist — shared target-browser config, extracted from Autoprefixer. https://github.com/browserslist/browserslist
[^4]: Autoprefixer README, "Grid Autoplacement support in IE". https://github.com/postcss/autoprefixer#grid-autoplacement-support-in-ie
[^5]: Browserslist, "update-db" — refreshing caniuse-lite. https://github.com/browserslist/update-db
[^6]: Autoprefixer 10.0.0 release notes. https://github.com/postcss/autoprefixer/releases/tag/10.0.0

## Tags

javascript, css, postcss, postcss-plugin, vendor-prefixes, browserslist, build-tools, frontend, caniuse, code-transformation
