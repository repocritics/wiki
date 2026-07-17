# unifiedjs/unified

> The core of an ecosystem that parses text into syntax trees, transforms them with plugins, and serializes them back — markdown, HTML, and prose share one interface.

[GitHub repo](https://github.com/unifiedjs/unified) ·
[Official website](https://unifiedjs.com) ·
[License: MIT](https://github.com/unifiedjs/unified/blob/main/license)

## Overview

unified is a small interface for processing content as structured data. On its own it does almost nothing: you configure it with a *parser*, zero or more *transformers*, and a *compiler*, and it turns text into a syntax tree, lets plugins inspect and rewrite that tree, then serializes it back to text[^1]. The value is not the ~500-line core but the collective around it — 500+ packages, most authored or curated by Titus Wormer (wooorm), spanning three content ecosystems: remark (markdown, mdast), rehype (HTML, hast), and retext (natural language, nlcst)[^2].

The defining idea is a shared node format. Every tree is a [unist][] node — a plain object with a `type` field and, by convention, `children` and `position` — so utilities and plugins compose across content types. This is what lets you parse markdown, cross-compile it to an HTML tree, and stringify HTML from one pipeline. It is also the ecosystem's central tension: the power comes from learning the tree formats (mdast vs hast vs nlcst) and the plugin conventions, and that learning curve is steep relative to a one-shot library like marked. You rarely reach for `unified` directly for a single content type — you use `remark` or `rehype` — and you reach for the core only when bridging kinds of content or building tooling.

unified is the invisible engine under MDX, Gatsby, Docusaurus, Astro's markdown, Prettier's markdown formatting, and a large share of the JavaScript documentation toolchain. Most developers consume it transitively without ever importing it.

## Getting Started

The package is ESM-only and targets Node 16+[^3]:

```sh
npm install unified remark-parse remark-rehype rehype-stringify
```

```js
import rehypeStringify from 'rehype-stringify'
import remarkParse from 'remark-parse'
import remarkRehype from 'remark-rehype'
import {unified} from 'unified'

const file = await unified()
  .use(remarkParse)      // text -> mdast (parser)
  .use(remarkRehype)     // mdast -> hast (transformer, "mutate" bridge)
  .use(rehypeStringify)  // hast -> HTML (compiler)
  .process('# Hello world!')

console.log(String(file)) // => <h1>Hello world!</h1>
```

There is no default export; `unified` is a named export and is itself a frozen root processor you call to create a working one.

## Architecture / How It Works

A processor runs three phases: **parse** (text to tree via the `parser`), **run** (the tree through each registered `transformer`, sync or async), and **stringify** (tree back to text via the `compiler`). `.process()` does all three; `.parse()`, `.run()`, and `.stringify()` expose them individually. Plugins are registered with `.use()` and are just functions that may attach a parser/compiler or return a transformer.

The tree formats are specified independently of unified in the [unist][] family: mdast (markdown), hast (HTML), nlcst (natural language), plus xast (XML) and esast (JavaScript). The `syntax-tree` GitHub org maintains hundreds of low-level utilities (`unist-util-visit`, `mdast-util-to-string`, `hast-util-sanitize`, and so on) that plugins are built from. `unified` core knows nothing about any specific format — it only orchestrates parser, transformers, and compiler.

Files are carried as [vfile][] objects, which accumulate the value, metadata, and lint-style messages (`file.messages`) that plugins emit. `vfile-reporter` renders those messages; this is how remark-lint and retext surface warnings with line/column positions.

Two subtle mechanisms trip people up:

- **Freezing.** Processors exposed from modules (`remark`, `rehype`) are frozen so importing them and mutating shared state can't leak across consumers. Calling `.use()` on a frozen processor throws; you must call the processor first (`rehype()` not `rehype`) to get a fresh, unfrozen copy[^4].
- **Bridge vs mutate.** Cross-ecosystem plugins run in one of two modes. *Mutate* (e.g. `remark-rehype`) replaces the tree and the pipeline continues on the new format. *Bridge* (e.g. `remark-retext`) runs a secondary processor for its side effects, then continues on the original tree. Confusing the two is a common source of "my later plugins see the wrong tree" bugs.

For tooling, `unified-engine` adds file-system discovery, config files, and ignore handling; `unified-args` builds CLIs on top of it; `unified-language-server` builds LSP servers. remark-cli and the rehype CLI are thin wrappers over these.

## Production Notes

**ESM-only is the biggest migration cost.** Since v10 the core and the entire ecosystem dropped CommonJS[^3]. Projects on `require()` cannot `import` these packages synchronously; the pinned-old-version escape hatch (staying on unified 9 / remark 13) means missing years of fixes. Jest and ts-node setups frequently need explicit ESM configuration or a dynamic `import()` shim. This is the single most reported friction point.

**Version alignment across the collective matters.** Because plugins share tree formats and helper types, mixing a new `remark-parse` with an old `mdast-util-*` (or a rehype plugin built against an older hast) can produce type errors or silently wrong trees. Upgrade an ecosystem's packages together, not piecemeal.

**Security is not automatic.** rehype produces HTML from arbitrary input; without `rehype-sanitize` you have an XSS vector. `remark-rehype` passes raw HTML through as a `raw` node that only becomes real HTML if you add `rehype-raw` — a step that both enables desired behavior and re-opens the injection surface. Sanitize after raw handling, not before.

**Performance.** The full parse → transform → stringify pipeline builds and walks complete ASTs, so it is meaningfully heavier than a streaming or regex converter for the "just turn this markdown into HTML" case. Repeated `.use()` chains rebuild configuration; for hot paths, `.freeze()` a configured processor once and reuse it. The markdown parser (micromark, under remark) is CommonMark-compliant and correctness-first, which is the right default but not the fastest option available.

**Types.** unified ships types generated from JSDoc and leans heavily on generics to thread the tree format through the pipeline. The types are precise but can produce dense, hard-to-read errors when a plugin in the chain doesn't line up — often the real signal that ecosystem versions are mismatched.

## When to Use / When Not

**Use when:**
- You need to inspect or transform content as an AST, not just convert it once.
- You are bridging content types (markdown ⇄ HTML ⇄ prose) in one pipeline.
- You want a plugin ecosystem for linting, rewriting, or enriching markdown/HTML.
- You are building tooling: a static-site generator, a formatter, a linter, an LSP.

**Avoid when:**
- You only need a fast one-shot markdown-to-HTML conversion with no AST access.
- You are locked to CommonJS and cannot adopt ESM.
- Bundle size or cold-start weight is critical and you need no plugins.
- Your content is one fixed format and a single-purpose library already covers it.

## Alternatives

- markedjs/marked — reach for it when you want the smallest, fastest one-shot markdown→HTML converter and never need the syntax tree.
- markdown-it/markdown-it — use it for extensible markdown→HTML with a token-stream plugin API, when you don't need a reusable, cross-tool AST.
- postcss/postcss — the same architectural shape (plugins over a parsed tree) but scoped to CSS; use it when the content is stylesheets.
- prettier/prettier — use it when the goal is opinionated multi-language formatting rather than building your own transforms (it uses remark internally for markdown).
- babel/babel — use it when the AST you need is JavaScript/TypeScript source specifically.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-07 | Repository created; unified factored out of the retext/remark line as a shared processor[^5]. |
| 6–9 | 2016–2021 | Plugin/preset API, `data()`, freeze semantics, and the bridge/mutate model stabilized (CommonJS era). |
| 10.0.0 | 2022 | ESM-only; CommonJS support dropped across the core and ecosystem[^3]. |
| 11.0.0 | 2023 | Current major; types reworked (JSDoc-generated), tightened generics for tree formats[^6]. |

## References

[^1]: unified README — "Overview": parser, transformers, compiler, and the process phases. https://github.com/unifiedjs/unified#overview
[^2]: unified README — "What is this?": collective of 500+ packages; remark, rehype, retext ecosystems. https://github.com/unifiedjs/unified#what-is-this
[^3]: unified README — "Install": ESM-only, Node 16+. https://github.com/unifiedjs/unified#install
[^4]: unified README — `processor.freeze()`: error thrown on `.use()` of a frozen processor. https://github.com/unifiedjs/unified#processorfreeze
[^5]: GitHub API — `unifiedjs/unified` repository `created_at` 2015-07-31. https://github.com/unifiedjs/unified
[^6]: unified changelog / releases. https://github.com/unifiedjs/unified/releases

## Tags

javascript, ast, syntax-tree, markdown, html, unist, remark, rehype, retext, content-processing, esm, parser
