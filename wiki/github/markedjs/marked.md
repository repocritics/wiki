# markedjs/marked

> A regex-based markdown parser and compiler for JavaScript, optimized for speed and small size rather than strict spec conformance.

[GitHub repo](https://github.com/markedjs/marked) ·
[Official website](https://marked.js.org) ·
[License: MIT](https://github.com/markedjs/marked/blob/master/LICENSE.md)

## Overview

Marked is one of the oldest and most-installed markdown-to-HTML libraries in the JavaScript ecosystem. It was written by Christopher Jeffrey (chjj) and first published around 2011; the repository was created in July 2011[^1]. Since 2018 it has been maintained by the MarkedJS organization rather than the original author, with Tony Brix (UziTech) as the long-running lead maintainer. It has no runtime dependencies and ships as ESM, CommonJS, and a UMD browser bundle.

The defining design choice is that Marked is **regex-based, not grammar-based**. It tokenizes markdown with a set of hand-tuned regular expressions in a block-level lexer and an inline lexer, then renders those tokens to HTML in a second pass. This makes it fast and tiny, but it means Marked approximates the CommonMark and GFM specifications rather than implementing them exactly. It tracks its conformance against the CommonMark and GFM test suites and openly reports that it is not 100% compliant — edge cases in nested constructs, HTML blocks, and list interactions can differ from the reference parsers. For most documentation and content-rendering use cases this is invisible; for anything that must round-trip or match CommonMark byte-for-byte it matters.

The second thing to internalize before using Marked: **it does not sanitize its output.** Marked emits whatever HTML the markdown implies, including raw `<script>` and event-handler attributes from inline HTML. The built-in `sanitize`/`sanitizer` options were deprecated and then removed; the project now explicitly tells you to run the output through DOMPurify or an equivalent[^2]. Treating Marked's output as safe HTML is the single most common security mistake made with it.

## Getting Started

```sh
npm install marked        # library
npm install -g marked     # CLI
```

```js
import { marked } from 'marked';
import DOMPurify from 'dompurify';

// Marked does NOT sanitize — always sanitize untrusted input's output.
const dirty = marked.parse('# Hello\n\n<img src=x onerror=alert(1)>');
const clean = DOMPurify.sanitize(dirty);
console.log(clean);
```

Command line, reading stdin:

```bash
echo '# hello world' | marked
# <h1>hello world</h1>
```

## Architecture / How It Works

The pipeline is two clean stages, both of which are independently callable:

1. **Lexer** (`marked.lexer()`) — a block-level tokenizer walks the source line region by region (headings, lists, blockquotes, code fences, tables, paragraphs), and an inline lexer tokenizes spans (emphasis, links, code spans, inline HTML) inside those blocks. The result is a flat/nested token array.
2. **Parser** (`marked.parser()`) — walks the tokens and calls a `Renderer` method per token type to produce HTML.

Because tokens are a first-class intermediate representation, you can call `marked.lexer(md)`, inspect or mutate the tokens, then `marked.parser(tokens)`. This is the seam most integrations use.

**Extensibility** goes through `marked.use()`[^3], which accepts:
- a custom **renderer** (override how a token type becomes HTML),
- a custom **tokenizer** (override or add block/inline parsing),
- **extensions** (self-contained `{ name, level, start, tokenizer, renderer }` objects for new syntax),
- **`walkTokens`** (a visitor run over every token — the standard place to post-process or fetch async data),
- **hooks** (`preprocess`, `postprocess`, `processAllTokens`) to wrap the whole run.

Marked is **synchronous by default**. Async work (e.g. syntax highlighting that returns a promise, or fetching content in `walkTokens`) requires setting the `async: true` option, which makes `marked.parse()` return a promise. Mixing async extensions without that flag is a frequent source of silently-dropped output.

The regex core is the reason for both its speed and its historical fragility. There is no AST in the CommonMark sense and no streaming parser; the whole document is held in memory and processed in two passes.

## Production Notes

- **Sanitize downstream, always.** There is no safe mode. Any path where user-authored markdown reaches a browser must pass Marked's output through DOMPurify (recommended), sanitize-html, or insane. Do not rely on removing the raw-HTML capability — Marked will still emit dangerous URLs and attributes.
- **ReDoS history.** Because parsing is regex-driven, Marked has shipped catastrophic-backtracking (ReDoS) vulnerabilities where crafted input pins a CPU. Notable examples were fixed in 4.0.10 (CVE-2022-21680, CVE-2022-21681)[^4]. If you render untrusted markdown on a server, keep Marked current and consider running it with a timeout/worker isolation. This class of bug is inherent to the regex approach and recurs.
- **Options changed meaning across majors.** Long-standing options were removed, not just deprecated: `sanitize`, `sanitizer`, `headerIds`, `headerPrefix`, `mangle`, `langPrefix` handling, and the `highlight` option have all been removed or relocated to extensions across the v5–v16 line. Header IDs and heading anchors now require `marked-gfm-heading-id`; syntax highlighting now requires `marked-highlight`. Upgrading across a major without reading its changelog will silently drop features you depended on.
- **Fast, breaking major cadence.** Marked follows semver strictly and cuts a major version whenever it drops an end-of-life Node.js version or removes a deprecated option. That produced five majors in 2023 alone (v5 through v9) and reached v18 by 2026. Pin a version and read release notes before bumping; "just take the latest" is not a safe upgrade policy here.
- **Node support is narrow.** Only current and active-LTS Node.js versions are supported; EOL Node may break at any release. The browser target is "Baseline Widely Available."
- **Not a transform tool.** Marked renders to HTML. If you need to analyze, lint, or rewrite markdown (or output anything other than HTML), its token array is workable but far less ergonomic than an AST library.

## When to Use / When Not

**Use when:**
- You need to turn trusted or sanitized markdown into HTML quickly with zero dependencies.
- Bundle size and raw speed matter more than perfect CommonMark conformance.
- You want a simple `parse()` call plus lightweight renderer/extension overrides.
- You're rendering docs, README content, or comments (with sanitization) in the browser or Node.

**Avoid when:**
- You need exact CommonMark/GFM conformance or must match another parser byte-for-byte.
- You want to transform markdown as a structured AST (linting, rewriting, MDX, multi-format output).
- You render untrusted input and cannot add a sanitizer and keep the library patched.
- You need a stable API that rarely breaks — Marked's major cadence is deliberately aggressive.

## Alternatives

- markdown-it — use instead when you want CommonMark conformance plus a mature plugin ecosystem (it powers VitePress and many editors); slightly larger and slower.
- remarkjs/remark — use instead when you need a real markdown AST for linting, transformation, or MDX via the unified ecosystem.
- micromark — use instead when you want a small, strictly CommonMark-compliant, streaming tokenizer (it is the engine under remark).
- commonmark/commonmark.js — use instead when you need the reference implementation and spec-exact output.
- showdownjs/showdown — use instead when you need bidirectional HTML↔markdown conversion in one older library.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-07 | Created by Christopher Jeffrey (chjj)[^1]. |
| 1.0.0 | 2020-04-21 | First 1.x under the MarkedJS org; API stabilization. |
| 2.0.0 | 2021-02-07 | Renderer/tokenizer extension refinements. |
| 3.0.0 | 2021-08-16 | `marked.use()` extension API matured. |
| 4.0.0 | 2021-11-02 | ESM-first build; UMD bundle for browsers. |
| 5.0.0 | 2023-05-02 | Removed deprecated `sanitize`/`mangle`/`headerIds` etc.[^2] |
| 6–9.0.0 | 2023 | Rapid majors dropping EOL Node and deprecated options. |
| 12.0.0 | 2024-02-03 | Continued Node-support-driven major bumps. |
| 15.0.0 | 2024-11-09 | — |
| 18.0.0 | 2026-04-07 | Latest major line (v18.0.6 as of 2026-07)[^5]. |

## References

[^1]: markedjs/marked repository, created 2011-07-24. https://github.com/markedjs/marked
[^2]: Marked usage docs — output is not sanitized; use DOMPurify. https://marked.js.org/using_advanced
[^3]: Marked extensibility docs (`marked.use`, renderer, tokenizer, walkTokens, hooks). https://marked.js.org/using_pro
[^4]: GitHub Advisory — ReDoS in marked, fixed in 4.0.10 (CVE-2022-21680 / CVE-2022-21681). https://github.com/markedjs/marked/security/advisories
[^5]: Marked releases (v18.0.6, published 2026-07-09). https://github.com/markedjs/marked/releases

## Tags

javascript, markdown, parser, compiler, commonmark, gfm, html-rendering, browser, nodejs, zero-dependency, security-sanitization
