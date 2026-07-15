# markdown-it/markdown-it

> A CommonMark-compliant Markdown parser for JavaScript, built around a pluggable rule pipeline and a flat token stream rather than an AST.

[GitHub repo](https://github.com/markdown-it/markdown-it) ·
[Live demo](https://markdown-it.github.io) ·
[License: MIT](https://github.com/markdown-it/markdown-it/blob/master/LICENSE)

## Overview

markdown-it is a Markdown-to-HTML parser for Node.js and the browser. Its stated goal is full [CommonMark](https://spec.commonmark.org/) compliance plus a curated set of extensions (GFM tables, strikethrough, URL autolinking via linkify-it, and a "typographer" for smart quotes and replacements) that can each be toggled on or off[^1]. The project was created in December 2014 by Vitaly Puzrin and Alex Kocharin, who had written the bulk of the earlier *Remarkable* parser and moved to a new codebase under their own leadership — the authors describe it as a successor, not a fork[^2].

The defining design choice is that markdown-it does **not** produce a syntax tree. It emits a flat array of tokens with explicit `nesting` markers (`+1` open, `0` self-closing, `-1` close), and a separate renderer walks that array to produce HTML[^3]. This makes rendering fast and rule replacement straightforward, but it means anyone who wants tree-shaped transforms (the mental model behind remark/mdast) is working against the grain.

For most JavaScript projects that need "render Markdown to HTML, with a few extensions, and let me plug in more," markdown-it is the default. It sits between the minimalist speed-first parsers (marked) and the heavyweight AST toolchains (remark/unified), and has been stable and actively maintained for over a decade.

## Getting Started

```bash
npm install markdown-it
```

```js
import markdownit from 'markdown-it'

const md = markdownit()                       // "default" preset
const html = md.render('# markdown-it rulezz!')

// inline-only (no <p> wrapper)
md.renderInline('__markdown-it__ rulezz!')

// enable optional features + load a plugin
const md2 = markdownit({ html: false, linkify: true, typographer: true })
  .use(somePlugin, { /* opts */ })
```

Presets select rule/option bundles: `markdownit('commonmark')` for strict spec behavior, `markdownit('zero')` for everything disabled (opt back in with `.enable([...])`), or the default when the argument is omitted[^1].

## Architecture / How It Works

Parsing runs as three ordered **rulers**, each an ordered list of named rules operating on a mutable state object:

1. **core** (`parser_core`) — normalization, block/inline orchestration, and post-processing passes like `linkify`, `replacements`, and `smartquotes`.
2. **block** (`parser_block`) — splits the source into block tokens (paragraphs, headings, lists, fences, tables).
3. **inline** (`parser_inline`) — tokenizes inline content (emphasis, links, code spans, autolinks).

Each ruler exposes `md.core.ruler`, `md.block.ruler`, and `md.inline.ruler` with `.before()`, `.after()`, `.at()`, `.push()`, `.enable()`, and `.disable()`. Rules are plain functions; a plugin is just a function passed to `md.use()` that registers or replaces rules and renderer overrides[^3]. Because rules are addressed by name, a plugin can surgically swap the built-in `fence` renderer or insert a rule before `emphasis` without patching source.

Output is governed by `md.renderer.rules`, a map from token type to a render function. Overriding a single entry (e.g. adding `target="_blank"` to links) is the idiomatic customization; you rarely touch the token stream directly. The token model — flat list plus nesting depth — is deliberately not a tree, which keeps the renderer a simple loop but forces plugin authors doing structural rewrites to track balance manually.

State is carried by `StateCore`, `StateBlock`, and `StateInline` objects that hold the source, position, token array, and the `md` instance. Since 14.0 the library ships as ESM (`.mjs` sources with an `import` default), with a UMD bundle in `dist/` for browsers where it attaches as `window.markdownit`[^4].

## Production Notes

**Sanitization is your job.** markdown-it is "safe by default" only because `html: false` strips raw HTML tags. Set `html: true` and the parser passes raw HTML through untouched — it does **not** sanitize. The maintainers are explicit that output must be run through a sanitizer like DOMPurify if you accept untrusted input[^5]. This is the single most common security footgun in the ecosystem.

**The `highlight` callback must return escaped HTML.** If your highlighter returns a raw string, you reintroduce an injection vector; the documented pattern falls back to `md.utils.escapeHtml(str)`. Also, if `highlight`'s result starts with `<pre`, markdown-it skips its own wrapper — a subtle behavior when overriding fence rendering.

**Tokens, not trees.** Teams arriving from remark/unified repeatedly try to walk a tree that does not exist. Structural transforms (e.g. "wrap every image in a figure") mean iterating the flat token array and maintaining nesting counters yourself, or moving the work into a renderer rule.

**ESM migration (14.0) is a breakage line.** The move to ESM-only sources dropped older Node versions and changed how some CJS consumers and older plugins import the package. Pinning to 13.x is the usual escape hatch for legacy CommonJS toolchains that cannot yet consume ESM.

**Plugin quality varies.** The `.use()` ecosystem is large but uneven; plugins that reach into internal rule ordering can break across minor versions of markdown-it itself. Audit third-party plugins for how deeply they hook the ruler.

**Performance is good but not the headline.** The full-feature build is roughly 1.5× slower than a CommonMark-only configuration because of the extra passes (linkify, typographer)[^6]. It is fast enough for server-side rendering at scale; if raw throughput dominates, marked is lighter.

## When to Use / When Not

**Use when:**
- You need CommonMark compliance plus a few well-defined extensions, toggleable per instance.
- You want to customize output by overriding named rules/renderers rather than post-processing HTML.
- You're rendering to HTML and don't need a manipulable syntax tree.
- You want a mature, single-purpose dependency with a long maintenance track record.

**Avoid when:**
- You need to transform Markdown structurally (lint, restructure, convert to other formats) — the AST-based remark/unified stack is built for that.
- You want the absolute smallest/fastest renderer and can accept looser spec compliance — marked is leaner.
- You require guaranteed sanitized output out of the box — markdown-it does not sanitize; you must add DOMPurify or equivalent.

## Alternatives

- markedjs/marked — faster and smaller, simpler API; looser CommonMark compliance and a thinner extension model. Use when throughput and bundle size matter more than strict spec fidelity or deep plugins.
- remarkjs/remark — AST-based (mdast) parser in the unified ecosystem. Use when you need to lint, transform, or convert Markdown structurally rather than just render HTML.
- micromark/micromark — the low-level, spec-strict CommonMark tokenizer underneath remark. Use when you want streaming/compliance primitives and are willing to work at a lower level.
- commonmark/commonmark.js — the reference implementation. Use when you want canonical spec behavior with minimal extension surface.
- executablebooks/markdown-it-py — a faithful Python port of markdown-it. Use when you need the same rule/plugin model outside JavaScript.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014-12 | Initial release; authors split from Remarkable to lead a CommonMark-first parser[^2]. |
| 12.0 | 2021 | Modernization; dropped legacy Node versions[^7]. |
| 13.0 | 2022 | Maintenance line; CommonMark spec updates, still CommonJS-friendly[^7]. |
| 14.0 | 2023-12 | ESM migration: `.mjs` sources, `import` default, older Node dropped[^4][^7]. |

## References

[^1]: markdown-it README — features, presets, and options list. https://github.com/markdown-it/markdown-it/blob/master/README.md
[^2]: markdown-it README, "Authors" — successor to Remarkable, "not a fork." https://github.com/markdown-it/markdown-it#authors
[^3]: markdown-it developer docs — architecture, rulers, tokens, and plugins. https://github.com/markdown-it/markdown-it/tree/master/docs
[^4]: npm package `markdown-it` — ESM build and `dist/` UMD bundle. https://www.npmjs.com/package/markdown-it
[^5]: markdown-it docs, "Security" — no sanitization; use a sanitizer for untrusted input. https://github.com/markdown-it/markdown-it/blob/master/docs/security.md
[^6]: markdown-it README, "Benchmark" — full build ≈1.5× slower than CommonMark-only. https://github.com/markdown-it/markdown-it#benchmark
[^7]: markdown-it CHANGELOG. https://github.com/markdown-it/markdown-it/blob/master/CHANGELOG.md

## Tags

javascript, markdown, commonmark, parser, markdown-it, html-rendering, node, browser, plugin-architecture, text-processing
