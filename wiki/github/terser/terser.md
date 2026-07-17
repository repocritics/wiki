# terser/terser

> The default JavaScript minifier of the bundler era — AST-based compression
> and name mangling for ES6+, carried forward from the UglifyJS lineage.

[GitHub repo](https://github.com/terser/terser) ·
[Official website](https://terser.org) ·
[License: BSD-2-Clause](https://github.com/terser/terser/blob/master/LICENSE)

## Overview

Terser is a JavaScript parser, compressor, and mangler for ES6+ code. It was
forked in May 2018 from `uglify-es` — the ES6 branch of Mihai Bazon's UglifyJS2
— after its maintainer declared the branch unmaintained[^1]. The fork kept API
and CLI compatibility with `uglify-es` and `uglify-js@3`, which is why a decade
of UglifyJS documentation, options, and folklore still mostly applies.

Its position in the ecosystem is infrastructural rather than chosen: webpack's
production mode minifies through `terser-webpack-plugin` by default[^2], and
Rollup, Angular CLI, and many other pipelines call it under the hood. That is
how a 9.3k-star repo ends up with roughly 66 million npm downloads per week —
most users have never invoked it directly. The defining tension in 2026 is
speed versus output size: Terser is written in JavaScript and single-threaded,
so Rust/Go minifiers (esbuild, SWC, oxc) beat it on wall-clock time by one to
two orders of magnitude, while Terser still typically wins on final bytes by a
few percent. Vite made that tradeoff explicit by defaulting to esbuild and
keeping Terser as the opt-in "smaller but slower" choice[^3].

Development is active — releases continued through 5.49.0 (2026-07-08) and the
last push to master was days ago — but the bus factor is low: the project is
effectively carried by one primary maintainer with OpenCollective funding, and
the issue backlog (~345 open) reflects that bandwidth.

## Getting Started

```bash
npm install terser            # programmatic use
npx terser input.js --compress --mangle --output input.min.js
```

Programmatic API — `minify()` returns a Promise since v5[^4]:

```js
import { minify } from "terser";

const result = await minify(
  { "app.js": "const square = (x) => x * x; console.log(square(3));" },
  {
    compress: { passes: 2 },
    mangle: true,
    sourceMap: { filename: "app.min.js", url: "app.min.js.map" },
  }
);

console.log(result.code); // console.log(9); with default evaluate/inline
console.log(result.map);  // source map JSON string
```

## Architecture / How It Works

Terser is a classic three-stage AST pipeline, inherited structurally from
UglifyJS2:

1. **Parse** — its own recursive-descent parser produces a custom AST class
   hierarchy (`AST_Node` subclasses), not ESTree. Acorn can be substituted at
   parse time (`--parse acorn`) and SpiderMonkey-format AST JSON is accepted,
   but everything downstream operates on Terser's own node classes.
2. **Compress** — a tree transformer applying dozens of heuristic rewrites:
   constant folding (`evaluate`), dead-code elimination, function and variable
   inlining, sequence collapsing, conditional simplification. Compression is
   purely syntactic with scope analysis — there is no type inference. The
   `passes` option re-runs the whole transformer; 2–3 passes shave additional
   bytes at proportional CPU cost, with diminishing returns after that.
3. **Mangle + output** — scope-aware renaming of variables (and optionally
   properties) to short base54 names, then code generation with source-map
   emission (via `@jridgewell/source-map` in current v5).

Semantic assumptions are surfaced as options rather than inferred. The
`unsafe*` compress family assumes standard built-ins are untouched; `pure_funcs`
and `/*#__PURE__*/` annotations let bundlers assert side-effect freedom that
Terser cannot prove itself — this annotation convention, popularized by the
UglifyJS/Terser lineage, is now honored across the whole minifier ecosystem.
Importantly, Terser is **not a transpiler**: the `ecma` option only changes
which output syntax it may emit (e.g. rewriting to arrow functions for
`ecma: 2015`), it never down-levels newer syntax to older targets.

## Production Notes

**Speed is the operational cost.** Minification is routinely the longest step
of a webpack production build. `terser-webpack-plugin` parallelizes across
worker processes by default, which helps on multi-chunk builds but not on one
huge chunk. If build time hurts more than ~1–2% extra bundle size, switching
the minifier to esbuild or SWC (both supported as drop-in `minify` options in
terser-webpack-plugin) is the standard fix[^2][^5].

**`--mangle-props` is the classic footgun.** Property mangling breaks any code
that accesses properties by string (`obj["name"]`), serializes objects to JSON
consumed elsewhere, or crosses into DOM/third-party APIs. It is off by default
for good reason; teams that enable it need `regex`/`reserved` discipline and a
shared `--name-cache` file so names stay consistent across separately-minified
chunks. Symptom of getting this wrong: code that works in dev and fails only in
production with unhelpful single-letter stack traces.

**Function-name reliance.** Frameworks and DI containers that read
`Function.prototype.name` or `constructor.name` break under mangling; the
escape hatches are `--keep-fnames` / `--keep-classnames` (at a size cost).

**License comments.** Default `comments` behavior preserves `@license`/`/*!`
comments Closure-style; teams stripping all comments for size should verify
they are meeting OSS license notice obligations another way.

**Upgrade pains have been mild.** The only breaking major since 2019 is
v5.0.0 (2020), which made `minify()` async and dropped the sync API[^4] —
callers wrapping Terser (Grunt/Gulp plugins, custom scripts) had to adapt.
The 5.x line has now run for six years; regressions occasionally ship in minor
releases (compress transforms are heuristic and the input space is all of
JavaScript), so pinning and testing minified output in CI is prudent.

**GitHub reports the license as NOASSERTION** because the LICENSE file has a
custom header, but the text is standard BSD-2-Clause.

## When to Use / When Not

**Use when:**
- You need the smallest output bytes and can pay for it in build time — Terser
  generally still edges out esbuild/SWC by a few percent.
- You are on webpack: it is the zero-config default and the best-tested path.
- You need UglifyJS-compatible options, name caches, or property mangling —
  features the newer Rust/Go minifiers cover only partially.
- You minify ES6+ (the actual UglifyJS supports ES5 input only).

**Avoid when:**
- Build speed is the bottleneck: esbuild minifies orders of magnitude faster
  at a small size penalty[^5].
- You expect transpilation: Terser will not convert ES2020 syntax to ES5 —
  pair it with Babel/SWC, or use a tool that does both.
- You want type-driven whole-program optimization: that is Closure Compiler's
  territory, with its corresponding annotation burden.

## Alternatives

- evanw/esbuild — use instead when minification time dominates your build and
  ~1–2% larger output is acceptable.
- swc-project/swc — Rust minifier with Terser-compatible options; the default
  in Next.js's SWC toolchain.
- mishoo/UglifyJS — the ES5-only ancestor; only relevant for legacy ES5-input
  pipelines.
- google/closure-compiler — advanced mode achieves deeper optimization via
  type information, at the cost of strict code conventions.
- oxc-project/oxc — emerging Rust toolchain (Rolldown ecosystem) with a
  minifier aiming at Terser-level output at native speed; still maturing.

## History

| Version | Date | Notes |
|---------|------|-------|
| Fork / 1.0 | 2018-05 | Forked from unmaintained `uglify-es`; first npm publish 2018-05-12[^1]. |
| 4.0 | 2019-05-19 | First major under the terser org; webpack-era consolidation. |
| 5.0 | 2020-08-01 | `minify()` becomes async (Promise); sync API removed[^4]. |
| 5.16–5.31 | 2022–2024 | Steady minors: compress improvements, `/*@__MANGLE_PROP__*/`, ES2022+ syntax support. |
| 5.49.0 | 2026-07-08 | Current release; 5.x line unbroken for six years. |

## References

[^1]: Alex Lam, comment declaring `uglify-es` unmaintained — 2018-05-29. https://github.com/mishoo/UglifyJS2/issues/3156#issuecomment-392943058
[^2]: webpack, `terser-webpack-plugin` documentation (default production minifier; `minify` swappable to esbuild/SWC). https://webpack.js.org/plugins/terser-webpack-plugin/
[^3]: Vite docs, `build.minify` — esbuild default, "terser … when the smallest bundle size is needed". https://vite.dev/config/build-options#build-minify
[^4]: Terser CHANGELOG, v5.0.0 — async `minify()`. https://github.com/terser/terser/blob/master/CHANGELOG.md
[^5]: esbuild benchmark and FAQ on minification tradeoffs. https://esbuild.github.io/faq/#minified-code

## Tags

javascript, minifier, compressor, mangler, ast, build-tools, bundler-ecosystem, es6, uglifyjs, source-maps
