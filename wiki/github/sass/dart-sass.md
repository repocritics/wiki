# sass/dart-sass

> The reference implementation of the Sass CSS preprocessor, written in Dart and shipped to JS users as the `sass` npm package.

[GitHub repo](https://github.com/sass/dart-sass) ·
[Official website](https://sass-lang.com/dart-sass) ·
[License: MIT](https://github.com/sass/dart-sass/blob/main/LICENSE)

## Overview

Dart Sass is the canonical implementation of Sass, the oldest and still most
widely deployed CSS preprocessor. Sass the *language* dates to 2006 and was
originally implemented in Ruby[^1]; `dart-sass` is the third and now sole
official implementation, and since 2019 it is the one that defines what Sass
*is* — new language features land here first and the specification follows the
implementation. Despite the name, most users never touch Dart: the code is
compiled to JavaScript and published as the `sass` package on npm, which is
what bundlers, `sass-loader`, Vite, and framework toolchains actually invoke.

The defining tension is that Sass predates most of what modern CSS can now do
natively. Nesting, custom properties, `calc()`, container queries, and CSS
color functions have all shipped in browsers, eroding the original reasons to
reach for a preprocessor. The Sass team's response has been to lean into the
parts CSS still lacks — the module system (`@use`/`@forward`), mixins,
functions, loops, and build-time computation — while committing to strict CSS
compatibility, which means Sass occasionally makes *breaking* changes to stay
aligned with new CSS syntax[^2]. Dart Sass is actively maintained: the repo
sees frequent releases and was last pushed within days of this writing, with
~4.2k stars that undercount its reach because the npm package (`sass`) draws
tens of millions of weekly downloads independent of GitHub attention.

## Getting Started

```bash
# JS/npm users (the common case) — this is Dart Sass compiled to JS:
npm install --save-dev sass

# Or a standalone CLI, no Node required:
brew install sass/sass/sass      # macOS/Linux
choco install sass               # Windows
```

```scss
// styles.scss — @use, not @import
@use "sass:math";

$gap: 16px;

.card {
  padding: $gap;
  width: math.div(100%, 3);   // sass:math, not the deprecated / operator
  &__title { font-weight: 700; }   // nesting
}
```

```js
// Compile programmatically (modern JS API)
const sass = require('sass');
const { css } = sass.compile('styles.scss');   // sync; compileAsync() exists but is slower
```

## Architecture / How It Works

Dart Sass is a single codebase that ships in three shapes from one version
number: a native standalone executable (Dart VM + snapshot), a Dart library on
pub.dev, and a JavaScript build on npm. Because one version spans all three,
the major version may bump even when only one distribution has a breaking
change[^2]. The pipeline is a conventional compiler: tokenizer → parser
producing a Sass AST → evaluation (resolving variables, mixins, functions,
`@use` graph) → a CSS AST → serializer. Dart users can reach the AST and the
load-resolution APIs through the separately versioned [`sass_api`
package](https://pub.dev/packages/sass_api).

The JS story is the important one for most consumers. The `sass` npm package is
Dart-compiled-to-JS, which means it is pure JavaScript with no native addon —
this is why it "just works" across platforms where the old C++ `node-sass`
routinely failed to build. The cost is speed: dart2js output is slower than the
native VM. For performance-sensitive or polyglot hosts, Dart Sass also
implements the compiler half of the **Embedded Sass protocol**, a
protobuf-over-stdio interface (`sass --embedded`) that lets a host language run
the fast native compiler as a subprocess and define custom importers and
functions across the boundary[^3]. The `sass-embedded` npm package wraps this.

The module system (`@use`/`@forward`) is the architectural centerpiece. Unlike
the legacy `@import`, `@use` loads each file once, namespaces its members, and
keeps private members (`-`/`_` prefixed) out of the public surface — turning
Sass from textual inclusion into an actual module graph[^4]. Custom importers
plug into this graph, which is how framework integrations resolve `~package`
paths, virtual files, and in-browser loads (the browser build drops
filesystem-based `compile()` and requires a `compileString()` importer).

## Production Notes

- **`@import` is on death row.** It is deprecated in favor of `@use`/`@forward`
  and slated for removal; large legacy codebases that lean on global `@import`
  behavior (implicit cross-file variable/mixin visibility) need a real
  migration, not a find-and-replace. The `sass-migrator` tool automates most of
  it but manual namespace cleanup is common.
- **The legacy JS API (`render`/`renderSync`) is deprecated** and will be
  removed in Dart Sass 2.0.0. New code should use `compile`/`compileString` and
  their async variants. Note `compileAsync()` is *substantially slower* than
  sync `compile()` — counterintuitive for Node developers who reflexively reach
  for async.
- **Division changed.** The `/` operator as division is deprecated; use
  `math.div()`. Slash now increasingly means "separator" to match CSS. This
  silently changes output in old stylesheets and is one of the most common
  deprecation-warning floods on upgrade.
- **Deprecation warnings are the upgrade tax.** Sass's compatibility policy is
  to emit deprecation warnings for at least three months before a breaking
  change lands in a minor release[^2]. On big codebases these warnings can
  number in the thousands; budget time to triage them rather than suppress.
- **`node-sass`/LibSass are dead ends.** LibSass (the C++ implementation) and
  its `node-sass` wrapper were officially deprecated in 2020[^5] and never
  gained the module system or modern color functions. Any project still on
  `node-sass` should migrate to `sass`; the two are not feature-equivalent.
- **Jest breaks Sass.** Jest's default test environment breaks JavaScript
  `instanceof`, which Dart Sass's JS build relies on heavily; you must install
  `jest-environment-node-single-context` and set it as `testEnvironment`[^6].
- **UTF-8 only.** Dart Sass only accepts UTF-8 source documents.

## When to Use / When Not

**Use when:**
- You want mixins, functions, loops, and a real module system that native CSS
  still does not offer.
- You maintain a large design system where build-time computation and shared
  partials pay off.
- You need a dependency-free, cross-platform compiler — the JS build has no
  native-addon fragility.

**Avoid when:**
- Your needs are covered by native CSS nesting, custom properties, and `calc()`
  — a preprocessor adds a build step for shrinking benefit.
- You want plugin-driven, future-CSS transforms (autoprefixing, polyfills):
  that is PostCSS's domain, not Sass's.
- You want CSS-in-JS scoping semantics tied to components — reach for a
  component-scoped styling solution instead.

## Alternatives

- postcss/postcss — plugin-based CSS transformer; use instead when you want
  autoprefixing and future-CSS polyfills rather than a Sass language.
- less/less.js — the other classic preprocessor; use if you inherit a Less
  codebase, though it is far less actively developed.
- sass/node-sass — legacy LibSass wrapper; deprecated, migrate off it, not to
  it.
- tailwindlabs/tailwindcss — utility-first framework; use when you want to
  avoid authoring CSS/Sass at all.
- evanw/esbuild / vitejs/vite — bundlers that invoke `sass` internally; use
  when you want Sass compilation folded into an existing build rather than run
  standalone.

## History

| Version | Date | Notes |
|---------|------|-------|
| Sass (Ruby) | 2006 | Original language + implementation, Ruby[^1]. |
| dart-sass repo | 2016-10 | Dart rewrite begins[^7]. |
| 1.0.0 | 2018 | First stable Dart Sass release. |
| 1.23.0 | 2019-10 | Module system (`@use`/`@forward`) launches[^4]. |
| — | 2019-03 | Ruby Sass reaches end of life; Dart becomes canonical[^8]. |
| — | 2020-10 | LibSass officially deprecated[^5]. |
| 1.x (ongoing) | 2021–2026 | First-class `calc()`, CSS Color 4 functions, continued `@import`/legacy-API deprecations. |

## References

[^1]: Sass language history — Sass was created in 2006 by Hampton Catlin and
developed by Natalie Weizenbaum. https://sass-lang.com/documentation/
[^2]: Compatibility policy — deprecation-then-break process and cross-distribution
versioning. https://github.com/sass/dart-sass#compatibility-policy
[^3]: Embedded Sass protocol specification.
https://github.com/sass/sass/blob/main/spec/embedded-protocol.md
[^4]: "The Module System is Launched" — Dart Sass 1.23.0.
https://sass-lang.com/blog/the-module-system-is-launched/
[^5]: "LibSass is Deprecated." https://sass-lang.com/blog/libsass-is-deprecated/
[^6]: Dart Sass README, "Using Sass with Jest."
https://github.com/sass/dart-sass#using-sass-with-jest
[^7]: sass/dart-sass repository, created 2016-10-31.
https://github.com/sass/dart-sass
[^8]: "Ruby Sass has reached End of Life."
https://sass-lang.com/blog/ruby-sass-is-deprecated/

## Tags

css-preprocessor, sass, scss, dart, css, build-tooling, frontend, compiler, npm-package, styling
