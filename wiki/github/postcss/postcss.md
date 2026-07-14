# postcss/postcss

> A CSS-to-AST transformation core that does almost nothing by itself — the value lives entirely in its plugins.

[GitHub repo](https://github.com/postcss/postcss) ·
[Official website](https://postcss.org) ·
[License: MIT](https://github.com/postcss/postcss/blob/main/LICENSE)

## Overview

PostCSS parses CSS into an abstract syntax tree, hands that tree to a chain of JavaScript plugins, and stringifies the result back to CSS[^1]. Created by Andrey Sitnik (author of Autoprefixer and Browserslist), it is not a preprocessor, a linter, or an autoprefixer — it is the shared runtime those tools are built on. On its own, `postcss([]).process(css)` returns the input essentially unchanged. Everything user-visible — vendor prefixing, nesting, minification, future-CSS transpilation, linting — is a third-party plugin.

This is the framework's defining tension. PostCSS core is small, stable, and maintained; the ~200+ plugin ecosystem around it is uneven in quality and lifespan[^2]. A "PostCSS setup" is really a curated list of plugins in a specific order, and most operational pain comes from that list, not from the core. The other structural fact worth knowing: PostCSS became load-bearing infrastructure largely because Autoprefixer, Stylelint, cssnano, and (through v3) Tailwind CSS all sit on top of it — so most projects depend on PostCSS transitively without ever writing a `postcss.config.js`.

## Getting Started

```bash
npm install --save-dev postcss autoprefixer postcss-cli
```

```js
// postcss.config.js — auto-discovered by most build tools (Vite, webpack, Parcel)
/** @type {import('postcss-load-config').Config} */
module.exports = {
  plugins: [
    require('autoprefixer'),        // vendor prefixes from Browserslist data
    require('postcss-nested'),      // unwrap Sass-style nesting
  ],
}
```

```js
// Direct JS API — plugins run in array order
const postcss = require('postcss')
const result = await postcss([require('autoprefixer')])
  .process('a { display: flex }', { from: 'src.css', to: 'out.css' })
console.log(result.css) // includes -webkit- prefixes per your browserslist
```

## Architecture / How It Works

Core is four pieces: a hand-written **tokenizer**, a **parser** that builds an AST, a **stringifier**, and a plugin runner (`LazyResult`) that walks the tree. The AST is a small node hierarchy — `Root` → `Rule` / `AtRule` / `Declaration` / `Comment` — and plugins mutate it in place. Nodes carry source positions, which is what makes accurate source maps and precise linter error locations possible.

Plugins come in two shapes. The legacy shape walks the tree itself (`root.walkRules(...)`). The **visitor API**, introduced in PostCSS 8 (2020), lets a plugin declare handlers like `Declaration(decl)` or `AtRule(atRule)` and the core visits nodes once, sharing a single traversal across all plugins[^3]. The 8.0 rewrite of this contract is the single most consequential event in PostCSS's history: every plugin had to be updated, and the `postcss` package became a strict peer dependency rather than a bundled one.

Because plugins are just functions over a shared mutable tree, **order is semantics**. `postcss-nested` must run before `autoprefixer`; a minifier must run last. There is no dependency graph or scheduler — the array you write is the pipeline, and getting it wrong fails silently (wrong output, not an error). This is the core's deliberate minimalism: it refuses to be opinionated, and pushes all opinion into the plugin list.

PostCSS ships TypeScript type definitions and can transform non-CSS syntaxes (SCSS, Less, `<style>` blocks, CSS-in-JS template literals) by swapping in a custom `parser`/`stringifier` — those variants only manipulate the AST, they do not compile Sass or Less to CSS[^1].

## Production Notes

**Peer-dependency duplication is the classic footgun.** Because `postcss` is a peer dependency of plugins, a lockfile can end up with both `postcss@7` and `postcss@8` installed. Plugins resolving the wrong copy produce cryptic "Plugin is not compatible / did you forget to await" errors. Dedupe and pin to a single recent `8.4.x`.

**Plugin abandonment is a real supply-chain risk.** Many popular plugins are single-maintainer and some never migrated past the PostCSS 7 API. Prefer plugins bundled into `postcss-preset-env` or the `csstools` monorepo, which are maintained together, over one-off plugins that solve a single property.

**Security.** PostCSS's parser has had multiple ReDoS advisories over the years; malicious or malformed CSS could hang the parser or, in one advisory, be mis-parsed in ways that let content bypass a downstream linter[^4]. These are fixed in recent releases — keep PostCSS updated rather than pinning an old minor, especially when it processes untrusted stylesheets.

**Performance.** Core parsing is fast; slowness is almost always a plugin. `postcss-preset-env` at a low target enables dozens of sub-plugins and can dominate build time. Autoprefixer is cheap but re-reads Browserslist — cache it in watch mode. When speed matters more than plugin breadth, teams increasingly reach for a Rust tool (see Alternatives).

**Ecosystem shift.** Tailwind CSS through v3 was a PostCSS plugin and drove much of PostCSS's install base; Tailwind v4 (2025) moved to its own engine and no longer requires PostCSS for its core path (a `@tailwindcss/postcss` adapter still exists)[^5]. Autoprefixer + cssnano remain the durable reasons most pipelines still include PostCSS.

## When to Use / When Not

**Use when:**
- You need Autoprefixer, cssnano, or Stylelint — you are already using PostCSS.
- You want to author future CSS today via `postcss-preset-env` with a Browserslist target.
- You need a custom, project-specific CSS transform and want a stable AST API to write it against.
- Your bundler (Vite, webpack, Parcel) already picks up `postcss.config.js` for free.

**Avoid when:**
- You want an all-in-one, fast, batteries-included transformer and don't want to curate a plugin list — Lightning CSS covers prefixing + minification + modern-syntax lowering in one Rust binary.
- You want a full preprocessor language (variables, mixins, functions, control flow) — that is Sass/Less, not PostCSS.
- You are on Tailwind v4's default pipeline and have no other PostCSS plugins — you may not need it at all.

## Alternatives

- parcel-bundler/lightningcss — Rust transformer/minifier that folds autoprefixer + cssnano + modern-syntax lowering into one fast tool; use when build speed matters and you don't need arbitrary plugins.
- sass/dart-sass — use when you want a real preprocessor language (Sass), not an AST transform toolkit.
- less/less.js — use when a project or theme system is already committed to Less syntax.
- tailwindlabs/tailwindcss — use when utility-first authoring is the goal; v4 supplies its own engine and largely sidesteps PostCSS.
- csstools/postcss-preset-env — not a replacement but the maintained meta-plugin most PostCSS users actually want instead of assembling future-CSS plugins by hand.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-09-24 | Repository created by Andrey Sitnik[^1]. |
| 1.0 | 2015 | First stable release; AST + plugin API established. |
| 8.0 | 2020-09 | Visitor plugin API; single shared traversal; `postcss` becomes a strict peer dependency — ecosystem-wide plugin migration[^3]. |
| 8.4 | 2021-11 | Current maintained line; successive `8.4.x` patches address ReDoS advisories[^4]. |

## References

[^1]: PostCSS README and API docs — "PostCSS is a tool for transforming styles with JS plugins." https://postcss.org/api/
[^2]: PostCSS plugins list (200+ community plugins). https://github.com/postcss/postcss/blob/main/docs/plugins.md
[^3]: PostCSS 8 release / visitor API and plugin migration guide. https://github.com/postcss/postcss/releases/tag/8.0.0 and https://evilmartians.com/chronicles/postcss-8-plugin-migration
[^4]: PostCSS security advisories (ReDoS / parsing), e.g. GHSA-7fh5-64p2-3v2j (CVE-2023-44270). https://github.com/postcss/postcss/security/advisories
[^5]: Tailwind CSS v4 — new engine, PostCSS no longer required on the default path. https://tailwindcss.com/blog/tailwindcss-v4

## Tags

css, postcss, ast, parser, css-transformation, build-tools, autoprefixer, javascript, nodejs, frontend, plugin-architecture, stylesheets
