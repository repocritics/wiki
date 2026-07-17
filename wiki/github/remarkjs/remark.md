# remarkjs/remark

> A markdown processor built as an ecosystem of plugins over a shared syntax tree — not a markdown-to-HTML function.

[GitHub repo](https://github.com/remarkjs/remark) ·
[Official website](https://remark.js.org) ·
[License: MIT](https://github.com/remarkjs/remark/blob/main/license)

## Overview

remark is the markdown half of the unified collective, maintained largely by Titus Wormer (wooorm)[^1]. The repository dates to 2014, when it was originally named `mdast`; the tool was later renamed to remark and the `mdast` name was kept for the abstract syntax tree it operates on[^2]. It is one of the most-depended-upon markdown parsers in the JavaScript ecosystem, sitting underneath MDX, Docusaurus, Gatsby, and Prettier's markdown formatter, among others.

The defining design choice is that remark does not turn markdown into HTML. It parses markdown into an AST (mdast), lets plugins inspect and mutate that tree, and serializes back to markdown. Producing HTML is a separate concern handled by crossing into the rehype (HTML) ecosystem via `remark-rehype`. This indirection is remark's central tradeoff: it is more code and more concepts than calling `marked(text)`, and in exchange you get a structured, inspectable, transformable representation and a plugin ecosystem of 150+ packages[^3].

The repository is a small monorepo of four packages: `remark-parse` (markdown → mdast), `remark-stringify` (mdast → markdown), `remark` (the two bundled with unified, for markdown-in/markdown-out), and `remark-cli` (a command-line wrapper). Nearly all interesting behavior — GFM, frontmatter, linting, tables of contents — lives in external plugins, not in this repo.

## Getting Started

```sh
npm install unified remark-parse remark-rehype rehype-stringify
```

```js
import rehypeStringify from 'rehype-stringify'
import remarkParse from 'remark-parse'
import remarkRehype from 'remark-rehype'
import {unified} from 'unified'

const file = await unified()
  .use(remarkParse)       // markdown -> mdast
  .use(remarkRehype)      // mdast -> hast (HTML AST)
  .use(rehypeStringify)   // hast -> HTML string
  .process('# Hello, *Mercury*!')

console.log(String(file)) // => '<h1>Hello, <em>Mercury</em>!</h1>'
```

To transform markdown and emit markdown (e.g. a codemod over docs), use the bundled `remark` package with `remark-stringify` built in, and walk the tree with `unist-util-visit`.

## Architecture / How It Works

The pipeline is: **micromark** tokenizes and parses markdown, `mdast-util-from-markdown` builds the mdast tree, plugins run as tree transformers, and `mdast-util-to-markdown` serializes. `remark-parse` and `remark-stringify` are thin adapters that wire those utilities into unified's processor. unified itself is the generic engine — a `.use(plugin)` chain, a shared `VFile` for content plus messages, and a three-phase run (parse → transform → stringify)[^4].

The parser was rewritten on top of micromark around remark 13 (late 2020). The old hand-written parser had long-standing CommonMark edge-case failures; the micromark rewrite brought the parser to 100% CommonMark compliance and moved the tokenizer into a separately maintained, state-machine-based package[^5]. This is the most consequential internal change in the project's history and the reason older tutorials that reference the pre-13 parser internals are misleading.

mdast is a plain-JSON tree (`type`, `children`, `value`, plus `position` offsets). Because it is just data, plugins are ordinary functions returning a transformer `(tree, file) => void`. Syntax extensions (GFM, math, frontmatter, MDX) are not plugins in the tree-transform sense alone — they hook into micromark's tokenizer and the from/to-markdown utilities, which is why a package like `remark-gfm` ships tokenizer extensions rather than just tree edits.

The coupling story: remark is inseparable from unified, micromark, and the syntax-tree utility packages. You rarely install "just remark"; you install a constellation of small packages from three GitHub orgs (`remarkjs`, `unifiedjs`, `syntax-tree`, `micromark`) that version and evolve together.

## Production Notes

**ESM-only.** remark 14 and the surrounding unified 10 generation dropped CommonJS entirely — the packages are pure ESM and cannot be `require()`d[^6]. This is the single largest source of upgrade pain. CommonJS codebases, older Jest setups, and tools that don't understand ESM need a dynamic `import()`, a bundler, or to stay on remark 13. Teams unaware of this hit `ERR_REQUIRE_ESM` at upgrade time.

**Plugin order matters and is silent.** Because plugins mutate a shared tree, ordering bugs are common: run `remark-gfm` before a plugin that reads GFM nodes, sanitize before stringify, and so on. There is no type-level enforcement of ordering; a misordered chain usually produces wrong output, not an error.

**Security.** Markdown → HTML is not safe by default. remark preserves raw HTML in the tree; if you convert to HTML you must run `rehype-sanitize` or you inherit an XSS surface. remark also warns about DoS from pathological inputs (thousands of nested lists/links, or very large files); the maintainers recommend capping input size and processing in a worker so it can be aborted[^3].

**Round-trip is normalizing, not lossless.** `remark-stringify` reformats markdown to its own canonical style (bullet character, emphasis marker, fence style, list indentation). Passing a document through parse→stringify will rewrite formatting even where you changed nothing — which is the point when using `remark-cli --output` as a formatter, but a surprise if you expected byte-preservation. Prettier's markdown printer is built on this behavior.

**Node support.** The unified collective tracks maintained Node.js versions and drops old ones on majors; recent lines target Node 16+[^7]. Pin ranges accordingly in CI.

**Types via JSDoc.** The org ships types generated from JSDoc rather than `.ts` sources; `@types/mdast` provides the tree types. Writing a correctly typed plugin requires importing `Root` from `mdast` and annotating the transformer, which is under-emphasized in most third-party tutorials.

## When to Use / When Not

**Use when:**
- You need to programmatically inspect or transform markdown (linting, codemods, ToC generation, link rewriting), not just render it.
- You're building on MDX, Docusaurus, or Gatsby, which already sit on remark.
- You want a plugin ecosystem and a stable, spec-compliant AST you can target.

**Avoid when:**
- You only need markdown → HTML once, quickly, with minimal dependencies — `marked` or `markdown-it` are a single package and simpler mental model.
- You're locked to CommonJS and cannot adopt ESM.
- You want the smallest possible compliant core with no AST — use `micromark` directly.

## Alternatives

- markedjs/marked — use when you want a tiny, fast markdown-to-HTML function with no AST or plugin architecture.
- markdown-it/markdown-it — use when you want fast HTML output and a mature plugin system but don't need a reusable, serializable tree.
- micromark/micromark — use when you only need compliant markdown→HTML and want the smallest core; it is remark's own tokenizer.
- mdx-js/mdx — use when you want JSX embedded in markdown; it is built on top of remark.
- commonmark/commonmark.js — use when you want the CommonMark spec's canonical reference implementation.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-07 | Repository created as `mdast`, later renamed to remark[^2]. |
| remark 13 | 2020-11 | Parser rewritten on micromark; 100% CommonMark compliance[^5]. |
| remark 14 | 2022 | ESM-only; CommonJS support dropped alongside unified 10[^6]. |
| remark 15 | 2023 | Current major line; continued mdast/micromark utility alignment. |

## References

[^1]: remark license and authorship — MIT © Titus Wormer. https://github.com/remarkjs/remark/blob/main/license
[^2]: unified / syntax-tree naming history; mdast is the markdown AST used by remark. https://github.com/syntax-tree/mdast
[^3]: remark README — plugins ecosystem and security guidance. https://github.com/remarkjs/remark#security
[^4]: unified — the content-transformation engine remark plugs into. https://github.com/unifiedjs/unified
[^5]: micromark — CommonMark-compliant tokenizer used by remark since v13. https://github.com/micromark/micromark
[^6]: unified collective ESM migration ("Please upgrade to ESM"). https://github.com/remarkjs/remark/issues/622#issuecomment-703019823
[^7]: remark README — compatibility with maintained Node.js versions. https://github.com/remarkjs/remark#compatibility

## Tags

javascript, markdown, parser, ast, commonmark, unified, plugins, esm, mdast, static-site
