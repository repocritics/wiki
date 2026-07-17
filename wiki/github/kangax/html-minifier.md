# kangax/html-minifier

> The long-standing reference JavaScript HTML minifier — highly configurable,
> well-tested, and now officially unmaintained.

[GitHub repo](https://github.com/kangax/html-minifier) ·
[Official website](http://kangax.github.io/html-minifier/) ·
[License: MIT](https://github.com/kangax/html-minifier/blob/gh-pages/LICENSE)

## Overview

HTMLMinifier is a JavaScript-based HTML compressor started by Juriy "kangax"
Zaytsev in 2010, introduced with a blog post that still doubles as the
project's design document[^1]. Its approach is parse-then-serialize: markup is
parsed into a tree, transformations are applied (whitespace collapsing,
comment stripping, attribute-quote removal, redundant-attribute pruning), and
minified markup is emitted. Every transformation is a separately toggleable
option, and nearly all of them are off by default — the tool's philosophy is
that aggressive minification is opt-in because several options can change
rendering or produce technically invalid HTML.

For most of the 2010s it was the de facto standard: `grunt-contrib-htmlmin`,
`gulp-htmlmin`, and `html-webpack-plugin` all wrapped it. That era is over.
The README's first line now states the project is no longer maintained and
directs users to HTML Minifier Next[^2]. The last npm release was v4.0.0 in
2019[^4]; the ~5,000 stars and 108 open issues reflect accumulated legacy
usage, not activity. Recent pushes to the repo (latest 2026-03) are
housekeeping such as the deprecation notice, not feature or fix work. This
page exists largely because the package still receives substantial npm
traffic through old build pipelines — and because choosing its successor is
the actual decision most readers face.

## Getting Started

```bash
npm install html-minifier   # note: unmaintained; see Alternatives
```

```js
const { minify } = require("html-minifier");

const result = minify('<p title="blah" id="moo">foo</p>', {
  collapseWhitespace: true,
  removeComments: true,
  removeAttributeQuotes: true,
  minifyCSS: true,   // delegates to clean-css
  minifyJS: true,    // delegates to UglifyJS (ES5 only)
});
// => '<p title=blah id=moo>foo</p>'
```

A CLI is included (`html-minifier --collapse-whitespace -o out.html in.html`).
Because everything defaults to off, calling `minify(input)` with no options
returns nearly unchanged output — a common first-use surprise.

## Architecture / How It Works

The core is a regex-driven, SAX-style HTML parser descended from John Resig's
2008 pure-JavaScript htmlparser[^6], modified for HTML5 rules (`html5: true`
by default). Parser events feed a serializer that applies the enabled
transformations as it re-emits markup. Consequences of this design:

- **Whole-document assumption.** The pipeline is tree in, tree out; the
  README is explicit that it cannot process invalid or partial chunks of
  markup. Template fragments must be excluded via `ignoreCustomFragments`
  (defaults handle `<% %>` and `<? ?>`) or `<!-- htmlmin:ignore -->` markers.
- **Delegated sub-minifiers.** `minifyCSS` calls clean-css, `minifyJS` calls
  UglifyJS, `minifyURLs` calls relateurl. The UglifyJS dependency is the
  project's most consequential fossil: it cannot parse ES2015+ syntax, so
  inline `<script>` with modern JavaScript breaks `minifyJS`. Replacing
  UglifyJS with terser was the founding purpose of the html-minifier-terser
  fork[^5].
- **Regex-extensible parsing.** Template-language quirks (Angular's
  `ng-click`, Handlebars conditionals inside tags) are handled by injecting
  user regexes via `customAttrAssign`, `customAttrSurround`, and
  `customEventAttributes` — flexible, but each regex widens the parser's
  attack and edge-case surface.
- **SVG special-casing.** SVG subtrees keep case-sensitivity and closing
  slashes regardless of global settings.

The project is developed on the `gh-pages` branch (source and demo site share
a branch), a period-typical oddity that confuses contributors expecting
`master`/`main`.

## Production Notes

- **It is abandoned; treat it as frozen.** No maintenance means no fixes: a
  reported ReDoS in v4.0.0 (CVE-2022-37620) remains unpatched and will flag
  in `npm audit`[^3]. For anything ingesting untrusted or third-party HTML,
  this alone disqualifies it.
- **`minifyJS` fails on modern JavaScript.** UglifyJS throws on ES2015+
  inline scripts. Symptoms are per-fragment errors or silently unminified
  scripts depending on configuration. Use html-minifier-terser.
- **Several options are unsafe by design.** `removeTagWhitespace` produces
  invalid HTML (documented); `collapseWhitespace` without
  `conservativeCollapse` can change rendering around inline elements;
  `removeOptionalTags` and `removeEmptyElements` can break scripts that
  query the DOM structurally. The blog post's option-by-option analysis[^1]
  is still the best guide to which are safe.
- **Performance is adequate, not competitive.** Single-threaded JavaScript
  string processing; large documents (multi-MB) take seconds. Rust-based
  minify-html is an order of magnitude faster on big corpora.
- **Ecosystem plugins have moved on.** `html-webpack-plugin` and modern
  toolchains switched to html-minifier-terser years ago; encountering
  `html-minifier` directly in a lockfile usually indicates a stale
  transitive dependency worth overriding.

## When to Use / When Not

**Use when:**
- You maintain a legacy pipeline already pinned to it, with trusted ES5-era
  input, and the migration cost outweighs the audit noise.
- You want its exhaustive option set as a reference design — its option
  taxonomy shaped every successor.

**Avoid when:**
- Starting anything new — use html-minifier-next or html-minifier-terser,
  which are drop-in API-compatible successors.
- Input includes ES2015+ inline scripts (UglifyJS limitation).
- Input is untrusted (unpatched ReDoS) or is a template fragment rather than
  a full document.
- Throughput matters at scale (Rust alternatives are far faster).

## Alternatives

- j9t/html-minifier-next — the officially designated successor; maintained
  fork with security fixes. Default choice for direct replacement[^2].
- terser/html-minifier-terser — the fork the bundler ecosystem adopted
  (html-webpack-plugin, vite plugins); use when you need terser-based
  `minifyJS` on modern JavaScript.
- wilsonzlin/minify-html — Rust core with Node/Python bindings; use when
  minification throughput on large volumes of HTML matters.
- posthtml/htmlnano — posthtml-plugin minifier; use when you already run a
  posthtml/Parcel pipeline.
- Swaagie/minimize — smaller, less configurable minifier aimed at
  server-side rendering; use for conservative minification with few knobs.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2010-05 | Initial release with the perfectionkills write-up[^1]. |
| 3.x | 2016–2017 | Most active era; rapid releases largely by co-maintainer Alex Lam (alexlamsl). |
| 4.0.0 | 2019 | Final npm release[^4]. Development effectively stops. |
| — | 2022 | CVE-2022-37620 (ReDoS) reported; never patched here[^3]. |
| — | 2025–2026 | README deprecation notice added, pointing to html-minifier-next[^2]. |

## References

[^1]: Juriy Zaytsev, "Experimenting with html-minifier" — 2010. http://perfectionkills.com/experimenting-with-html-minifier/
[^2]: html-minifier README deprecation notice, pointing to HTML Minifier Next. https://github.com/j9t/html-minifier-next
[^3]: CVE-2022-37620 — ReDoS in html-minifier ≤4.0.0. https://nvd.nist.gov/vuln/detail/CVE-2022-37620
[^4]: html-minifier on npm (latest: 4.0.0). https://www.npmjs.com/package/html-minifier
[^5]: html-minifier-terser — fork replacing UglifyJS with terser. https://github.com/terser/html-minifier-terser
[^6]: John Resig, "Pure JavaScript HTML Parser" — 2008. https://johnresig.com/blog/pure-javascript-html-parser/

## Tags

javascript, html, minifier, web-performance, build-tools, nodejs, cli, parser, unmaintained, legacy
