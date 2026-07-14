# rust-lang/mdBook

> A command-line tool that compiles a directory of Markdown files into a static HTML book — the engine behind The Rust Book and most of the Rust project's documentation.

[GitHub repo](https://github.com/rust-lang/mdBook) ·
[Official website](https://rust-lang.github.io/mdBook/) ·
[License: MPL-2.0](https://github.com/rust-lang/mdBook/blob/master/LICENSE)

## Overview

mdBook takes a `SUMMARY.md` table of contents plus a tree of Markdown chapters and produces a self-contained static website: a linear, book-shaped site with a sidebar, search, syntax highlighting, and theming. It was originally written by Mathieu David (azerupi) as a Rust reimplementation of the ideas in the then-JavaScript GitBook, and was adopted into the rust-lang organization once it became the build tool for official Rust documentation[^1]. It is what renders *The Rust Programming Language*, the Rust Reference, the Cargo book, the rustc dev guide, and a large fraction of the crate ecosystem's long-form docs.

The defining tradeoff is scope. mdBook deliberately models one shape of document — a single ordered book with nested chapters — and does that with a single static binary, no runtime, and near-instant builds. It is not a general static-site generator: there is one linear table of contents, no taxonomies, no page graph, no content collections. That constraint is the appeal (a `cargo install` and a `SUMMARY.md` gets you a publishable book in minutes) and the ceiling (anything that isn't book-shaped fights the tool).

Note the license: unlike the MIT-OR-Apache-2.0 default across most of the Rust ecosystem, mdBook is MPL-2.0[^2] — file-level weak copyleft. This rarely matters for using the tool, but it matters if you vendor or fork its source into a differently-licensed project.

## Getting Started

```bash
cargo install mdbook
# or: download a prebuilt binary from the GitHub releases page
```

```bash
mdbook init my-book        # scaffolds book.toml, src/SUMMARY.md, src/chapter_1.md
cd my-book
mdbook serve --open        # live-reloading dev server on http://localhost:3000
mdbook build               # renders static site into ./book/
```

The table of contents is a Markdown file with a specific structure:

```markdown
# Summary

[Introduction](./intro.md)

- [Getting Started](./getting-started.md)
  - [Installation](./install.md)
- [Reference](./reference.md)

[Appendix](./appendix.md)
```

Configuration lives in `book.toml`:

```toml
[book]
title = "My Book"
authors = ["Jane Doe"]
language = "en"

[output.html]
default-theme = "rust"
git-repository-url = "https://github.com/me/my-book"
```

## Architecture / How It Works

The build is a straight pipeline with two documented extension points:

1. **Load** — parse `book.toml`, then parse `SUMMARY.md` into an ordered chapter tree. `SUMMARY.md` is the source of truth; a chapter file that isn't listed there is not built.
2. **Preprocess** — the in-memory book (as JSON) is passed through a chain of *preprocessors*. Two are built in: `links` (expands `{{#include}}` / `{{#playground}}` directives) and `index` (turns `README.md` into `index.html`). Third-party preprocessors are separate executables invoked as subprocesses; mdBook writes the book as JSON to the process's stdin and reads the transformed book back from stdout.
3. **Render** — the processed book is handed to one or more *renderers* (backends). The default is the HTML backend; a `markdown` backend ships in-tree. Like preprocessors, custom renderers are external binaries that receive the book as JSON.

The HTML backend parses each chapter with **pulldown-cmark** (CommonMark, plus a GFM-ish subset — tables, strikethrough, task lists, footnotes), renders through **Handlebars** templates, and highlights code with **highlight.js** on the client. Search is built at compile time into a JSON index (an elasticlunr-compatible structure) that a small bundled JavaScript loads and queries entirely client-side — there is no server component. For Rust code blocks, the HTML backend injects a "Run" button wired to the Rust Playground, and `mdbook test` shells out to `rustdoc` to compile and run those same code samples as doctests.

The subprocess-and-JSON boundary is the whole plugin story. It keeps mdBook's core small and lets plugins be written in any language, at the cost of every plugin being an external binary you must install and keep on `PATH`.

## Production Notes

- **No incremental builds.** `mdbook build` re-renders the entire book every time. This is fine for hundreds of pages (builds are sub-second to a few seconds) but there is no partial-rebuild cache; `mdbook serve`'s watch mode simply reruns the full build on change.
- **`SUMMARY.md` is strict and unforgiving.** Structure is expressed purely through list nesting and link syntax; a stray indentation level or a malformed link produces a confusing parse error or a silently missing chapter. Draft chapters (title with no link) and separators have exact required syntax. This is the single most common source of "why isn't my page showing up" reports.
- **Plugins are external binaries.** Popular preprocessors (mdbook-mermaid, mdbook-katex, mdbook-toc, mdbook-admonish, mdbook-linkcheck) each require their own `cargo install` and must match a compatible mdBook version. CI that builds your book needs every one of them installed; a version skew between mdBook and a preprocessor can break the build after an upgrade.
- **Pin the mdBook version in CI.** The bundled HTML theme, its CSS/JS asset structure, and the search index format change across releases. Custom themes (`theme/` overrides) are coupled to internal template and class names, so upgrading mdBook can visually break a book that overrode theme files. Deploys on GitHub Pages / Netlify should pin an exact version rather than tracking latest.
- **`mdbook test` only covers Rust.** It runs code blocks through rustdoc; code samples in other languages are not executed or validated. Non-Rust books get no built-in code testing.
- **Search has limits.** The client-side index grows with book size and is loaded in full; very large books pay a download and memory cost, and there is limited relevance tuning compared to a hosted search service.
- **CommonMark, not arbitrary Markdown.** Raw-HTML-heavy content, or Markdown extensions pulldown-cmark doesn't implement, won't render as they might in other tools. Rich layout typically means a preprocessor or embedded HTML/CSS.

## When to Use / When Not

**Use when:**
- You're writing linear, chapter-structured long-form docs (a manual, a guide, a book).
- You want a single static binary, near-instant builds, and zero runtime to host on any static host.
- You're in the Rust ecosystem and want doctests, Playground integration, and consistency with official Rust docs.
- You want a low-dependency toolchain that a reviewer can `cargo install` and reproduce.

**Avoid when:**
- Your content isn't book-shaped — you need multiple content collections, tag/category taxonomies, blog indexes, or a landing site. Reach for a general SSG.
- You need a rich component/MDX authoring model, versioned docs sets, or i18n routing out of the box.
- Non-technical authors need a WYSIWYG/CMS editing surface (mdBook is files-in-Git only).
- You require server-side search or heavy interactive widgets beyond what a client-side static bundle allows.

## Alternatives

- squidfunk/mkdocs-material — Python/MkDocs with a far richer theme, navigation, and search; heavier toolchain, use when you outgrow the single-linear-book model.
- facebook/docusaurus — React/MDX docs sites with versioning and i18n; use when you need components, multiple doc versions, or a marketing site alongside docs.
- withastro/starlight — Astro-based docs framework; use when you want MDX and framework components with static output.
- GitBook — the hosted SaaS mdBook was originally modeled after; use when you want a managed WYSIWYG/hosting product rather than a Git-driven binary.
- asciidoctor/asciidoctor — use when your source is AsciiDoc and you need its stronger semantics for large technical manuals.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-07 | Repository created by Mathieu David (azerupi) as a Rust take on GitBook[^1]. |
| 0.1.x | 2017 | First stable-ish series; adopted under the rust-lang organization. |
| 0.2.0 | 2018 | Introduced the preprocessor + renderer plugin architecture (external-binary/JSON model). |
| 0.3.0 | 2019 | Search and configuration improvements; continued plugin ecosystem growth. |
| 0.4.0 | 2020-07 | Current major series; internal rewrite/cleanup and breaking config/theme changes. |

## References

[^1]: mdBook User Guide (official documentation and live demo of the tool). https://rust-lang.github.io/mdBook/
[^2]: License file, Mozilla Public License v2.0. https://github.com/rust-lang/mdBook/blob/master/LICENSE

## Tags

rust, documentation, static-site-generator, markdown, cli, books, ebook, docs-as-code, pulldown-cmark, mpl-2.0
