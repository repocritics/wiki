# rehypejs/rehype

> HTML as an abstract syntax tree — parse, transform with plugins, and stringify, built on the unified collective's hast tree.

[GitHub repo](https://github.com/rehypejs/rehype) ·
[Official website](https://unifiedjs.com) ·
[License: MIT](https://github.com/rehypejs/rehype/blob/main/license)

## Overview

rehype is the HTML arm of unified, the content-processing engine created by
Titus Wormer (wooorm)[^1]. Where its sibling remark handles Markdown, rehype
parses HTML into a tree, lets plugins inspect and mutate that tree, and
serializes it back to HTML. The tree format is hast (HTML AST)[^2], a
JSON-serializable structure shared across the syntax-tree ecosystem. rehype is
rarely used alone — its most common role is the HTML half of a Markdown-to-HTML
pipeline (`remark-rehype` bridges the two), which is how it ends up inside MDX,
Docusaurus, Astro's content layer, and most modern static-site toolchains.

This GitHub repository is a monorepo holding four small packages: `rehype-parse`
(HTML → hast), `rehype-stringify` (hast → HTML), `rehype` (both, glued together
with unified), and `rehype-cli` (a command-line wrapper). None of these do any
transformation on their own; the value lives in the plugin ecosystem —
`rehype-sanitize`, `rehype-highlight`, `rehype-slug`, `rehype-autolink-headings`,
and hundreds of others — most maintained outside this repo.

The defining tradeoff is composability versus weight. Treating HTML as data you
walk and edit with `unist-util-visit` is far more robust than regex or string
replacement, and the plugin model is genuinely reusable. The cost is a deep tree
of tiny single-purpose packages (unified, hast, vfile, dozens of `hast-util-*`
and `unist-util-*` helpers), a learning curve around the `.use()` pipeline, and
per-document parse/serialize overhead you would not pay with a template string.

## Getting Started

```bash
npm install rehype
# or the pieces: npm install unified rehype-parse rehype-stringify
```

```js
import {rehype} from 'rehype'

const file = await rehype()
  .use(function () {
    return function (tree) {
      // mutate the hast tree here
    }
  })
  .process('<!doctype html><title>Hi</title><h1>Hello</h1>')

console.log(String(file))
```

```js
// Fragment + a custom transform via unist-util-visit
import {unified} from 'unified'
import rehypeParse from 'rehype-parse'
import rehypeStringify from 'rehype-stringify'
import {visit} from 'unist-util-visit'

const file = await unified()
  .use(rehypeParse, {fragment: true})
  .use(() => (tree) => {
    visit(tree, 'element', (node) => {
      if (node.tagName === 'h1') node.tagName = 'h2'
    })
  })
  .use(rehypeStringify)
  .process('<h1>Hi, Saturn!</h1>')

console.log(String(file)) // <h2>Hi, Saturn!</h2>
```

## Architecture / How It Works

The pipeline is `unified()` at the core. A processor is a frozen, chainable
object: `.use(plugin, options)` registers a plugin, `.process(input)` runs it.
Plugins fall into two kinds — **parsers/compilers** (`rehype-parse` attaches a
parser, `rehype-stringify` attaches a compiler) and **transformers** (functions
that receive the tree and a vfile and mutate the tree in place). Parsing and
serialization each happen once; transformers run in registration order between
them.

The tree is hast. Every node has a `type` (`element`, `text`, `comment`,
`root`, `doctype`); elements carry `tagName`, `properties`, and `children`.
Property names use the DOM/JSX form — `className` (an array), `htmlFor` — not
the HTML attribute form, a frequent source of confusion. Traversal is not built
in; you pull in `unist-util-visit` and its siblings, the same utilities remark
uses, because hast is a specialization of the generic unist tree.

`rehype-parse` wraps `hast-util-from-html`, which uses `parse5` — the same
spec-compliant HTML parser jsdom relies on — so parsing matches browser behavior
including error recovery and implicit tags. `rehype-stringify` wraps
`hast-util-to-html`. The `fragment: true` option switches between whole-document
mode (adds `html`/`head`/`body` and doctype handling) and snippet mode.

The coupling story is the ecosystem itself: rehype is small, and almost
everything useful is a separate npm package with its own version and maintainer.
That keeps the core stable and lets you install only what you use, but a real
project's dependency tree includes unified, vfile, hast, and a long tail of
`*-util-*` micro-packages — shared across remark, retext, and rehype, so the
concepts transfer.

## Production Notes

- **Sanitize untrusted HTML, always.** rehype does not sanitize by default;
  parsing and re-serializing attacker-controlled HTML preserves `<script>`,
  event handlers, and `javascript:` URLs. Add `rehype-sanitize` (based on the
  GitHub schema) in any pipeline that touches user input[^3]. This is the single
  most common security mistake with the library.

- **ESM only.** Modern unified/rehype releases are pure ES modules; there is no
  `require()`. CommonWJS codebases must use dynamic `import()` or stay on older
  major versions. This has broken many upgrades and is the top compatibility
  friction point.

- **`raw` HTML in remark pipelines.** When bridging from Markdown with
  `remark-rehype`, embedded raw HTML is placed in the tree as opaque `raw` nodes
  and is only actually parsed if you add `rehype-raw`. Forgetting this produces
  output where inline HTML is escaped or dropped rather than rendered.

- **Property names bite.** hast uses the JSX property model (`className`,
  `htmlFor`, camelCased attributes), so transforms that assume HTML attribute
  names silently fail to match; consult `property-information` when in doubt.

- **Performance.** Each document is fully parsed into an object tree and
  serialized back — measurably heavier than templating for large HTML or
  high-throughput rendering. It is a transform tool, not a hot-path renderer.

- **Node support.** The collective targets maintained Node.js versions and drops
  unmaintained ones on each major; the current line aims to keep working on
  Node 16+[^4]. Pin versions if you need a fixed floor.

## When to Use / When Not

**Use when:**
- You need to programmatically inspect or rewrite HTML (add heading anchors,
  rewrite links, syntax-highlight code, sanitize).
- You are building a Markdown/MDX pipeline and need the HTML stage.
- You want composable, testable transforms instead of regex over markup.

**Avoid when:**
- You only need to generate HTML from data — a template engine (or JSX) is
  simpler and faster.
- You need a full live DOM with layout/events — use jsdom or a real browser.
- You want a single dependency with no ecosystem sprawl; rehype pulls a wide
  tree of micro-packages.

## Alternatives

- cheerio/cheerio — jQuery-style server-side HTML manipulation; more ergonomic
  for scraping and querying, less suited to reusable pipeline plugins.
- jsdom/jsdom — full DOM implementation; use when you need `document`, layout
  querying, or browser-API fidelity rather than a transform pipeline.
- posthtml/posthtml — plugin-based HTML transformer with a similar philosophy
  and a lighter tree; use when you want the plugin model without the unist
  ecosystem.
- remarkjs/remark — the Markdown sibling; use when your source is Markdown and
  you only bridge to HTML at the end.
- htmlparser2 (fb55) — a fast, low-level streaming parser; use when you want raw
  parsing speed and will build your own handling.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016 | rehype created within the unified collective, HTML support alongside remark[^1]. |
| unified 6+ | 2017–2019 | Consolidated processor API (`.use`/`.process`) shared across remark/rehype/retext. |
| ESM migration | ~2021 | Packages moved to pure ES modules, dropping CommonJS `require`. |
| current | 2026-06 | Actively maintained; latest push 2026-06-13, monorepo of parse/stringify/core/cli[^5]. |

## References

[^1]: unified collective and rehype, created by Titus Wormer. https://unifiedjs.com
[^2]: hast — HTML abstract syntax tree specification. https://github.com/syntax-tree/hast
[^3]: rehype-sanitize — make the tree safe against XSS. https://github.com/rehypejs/rehype-sanitize
[^4]: rehype README, "Compatibility" — targets maintained Node.js, aims to keep Node 16 support. https://github.com/rehypejs/rehype
[^5]: rehypejs/rehype repository metadata (stars, license MIT, last push), fetched via GitHub API 2026-07. https://github.com/rehypejs/rehype

## Tags

javascript, html, ast, unified, rehype, parser, html-transform, plugins, hast, ssg, static-analysis
