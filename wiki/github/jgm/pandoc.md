# jgm/pandoc

> The universal document converter — a Haskell library and CLI that translates between ~40 markup and word-processing formats through a single intermediate AST.

[GitHub repo](https://github.com/jgm/pandoc) ·
[Official website](https://pandoc.org) ·
[License: GPL-2.0-or-later](https://github.com/jgm/pandoc/blob/main/COPYRIGHT)

## Overview

Pandoc is a document converter written and maintained by John MacFarlane (jgm), a philosophy professor at UC Berkeley, with first commits going back to 2006[^1]. It reads a source format into an abstract syntax tree (the `Pandoc` type), then writes that tree out to a target format. Because every reader targets the same AST and every writer consumes it, adding one format gives you conversions to and from all the others — the combinatorial payoff that explains why one tool covers LaTeX, DOCX, EPUB, HTML, Markdown, reStructuredText, Org, JATS, ODT, and dozens more[^2].

The defining tension is right there in the README: the intermediate representation is deliberately *less expressive* than most formats it touches[^2]. Pandoc preserves document structure — headings, lists, tables, footnotes, citations, math — not presentation details like margins, fonts, or color. A round trip through the AST is lossy by design. Conversions *from* Pandoc's own Markdown aim to be faithful; conversions from richer formats (a heavily-styled DOCX, a complex LaTeX document) will drop what the AST cannot model. Understanding this is the difference between "pandoc is broken" and "pandoc did exactly what its data model allows."

Pandoc is the de facto backbone of academic and technical publishing pipelines: R Markdown, Quarto, Jupyter's `nbconvert`, mdBook-adjacent toolchains, and countless static-site and thesis workflows shell out to it. Its Markdown dialect — with tables, definition lists, citations via CSL, and math — is a reference point in its own right.

## Getting Started

Pandoc ships as a self-contained binary; most users never touch Haskell:

```bash
# macOS
brew install pandoc
# Debian/Ubuntu (distro versions lag; prefer the release .deb for current features)
sudo apt install pandoc
# or download a release from github.com/jgm/pandoc/releases
```

Basic conversion — format is inferred from file extensions, or set explicitly:

```bash
# Markdown to standalone HTML5
pandoc README.md -o readme.html

# Markdown to PDF (requires a LaTeX engine, or use --pdf-engine=weasyprint / typst)
pandoc paper.md --citeproc --bibliography=refs.bib -o paper.pdf

# Force formats when extensions are ambiguous, and read from stdin
pandoc -f docx -t gfm document.docx -o document.md
```

For PDF you must install an external engine — Pandoc does not render PDF itself; it drives LaTeX (default), Groff ms, `wkhtmltopdf`, `weasyprint`, or `typst`[^3].

## Architecture / How It Works

Pandoc's design is a readers-AST-writers pipeline, and the AST is the whole product. The core data type is a `Pandoc` value: document metadata plus a list of block elements (`Header`, `Para`, `CodeBlock`, `BulletList`, `Table`…), which contain inline elements (`Str`, `Emph`, `Link`, `Math`, `Cite`…). Every reader is a parser producing this tree; every writer is a function consuming it. The `native` and `json` formats expose the AST directly, which is how tooling and filters hook in.

**Filters** are the extensibility story. Because conversion routes through a stable AST, you can transform the tree between read and write. Two mechanisms exist: JSON filters (any language — the AST is piped as JSON to an external process) and Lua filters, which run in an embedded Lua interpreter inside pandoc with no process overhead and no serialization cost[^4]. Lua filters are the recommended path and power a large ecosystem (`pandoc-crossref`, diagram renderers, etc.). Custom readers and writers can also be written entirely in Lua, letting you add a format without recompiling.

**Citations** are handled by a built-in Haskell reimplementation of the Citation Style Language processor (`citeproc`), enabled with `--citeproc`. It reads bibliographies (BibTeX, CSL-JSON, RIS) and a CSL style file to render formatted references — this is why Pandoc is entrenched in academic writing.

The library is factored into several packages on Hackage: `pandoc-types` (the AST, versioned separately and the thing filters depend on), `texmath` (TeX/MathML/OMML math conversion), `citeproc`, `commonmark-hs` (the CommonMark parser), and `pandoc` itself. The AST version is significant: a filter or tool compiled against one `pandoc-types` major version can break against another, since the JSON shape changes.

## Production Notes

**PDF is not built in.** The single most common support question is a PDF failure that is really a LaTeX failure — a missing package, an unsupported Unicode glyph under pdfLaTeX (use `xelatex`/`lualatex`), or LaTeX not installed at all. Treat the PDF path as "pandoc emits LaTeX, then an external engine compiles it," and debug the engine separately with `--pdf-engine` and `-t latex` to inspect the intermediate.

**Distro packages are old.** Ubuntu/Debian and even some Homebrew-adjacent channels can lag the current release by many minor versions, and Pandoc adds formats and fixes format-specific bugs frequently. If a documented flag or format "doesn't exist," check `pandoc --version` before filing anything — install the official release binary instead of the distro package for anything current.

**Lossy conversion is expected, not a bug.** DOCX-to-Markdown will discard styling the AST can't represent; complex tables, text boxes, tracked changes, and equation edge cases are the usual casualties. For DOCX/ODT output, custom styling goes through a *reference document* (`--reference-doc=template.docx`) rather than CSS — you edit a template file's styles and pandoc maps AST elements onto them. HTML/LaTeX/EPUB output is controlled by templates (`--template`, `-V key=val`, partials) and standalone mode (`-s`).

**Markdown is not one thing.** Pandoc distinguishes `markdown` (its own extended dialect), `gfm`, `commonmark`, `commonmark_x`, `markdown_strict`, `markdown_mmd`, and `markdown_phpextra`, each with different extension sets toggled by `+ext`/`-ext` syntax. Mixing these up silently changes parsing behavior; be explicit about the flavor on both `-f` and `-t`.

**Performance and memory.** For most documents pandoc is fast, but very large single inputs are held and transformed in memory as a whole tree; there is no streaming. Batch pipelines that spawn pandoc per file are usually fine, but converting one enormous document (a full book in a single file) can be memory-heavy.

**Stability.** Releases are frequent and generally careful about backward compatibility for the CLI, but `pandoc-types` AST changes and occasional default/behavior changes to specific readers/writers can surprise filters and templates. Pin the pandoc version in reproducible pipelines (Quarto, CI) rather than tracking latest.

## When to Use / When Not

**Use when:**
- You need to convert between markup/word-processing formats and can accept a structure-preserving, presentation-lossy model.
- You're building an academic or technical publishing pipeline needing citations (CSL), math, cross-references, and multiple output targets from one source.
- You want programmatic document transformation via a stable AST and Lua/JSON filters.
- You need DOCX/ODT/EPUB/JATS output that most other Markdown tools can't produce.

**Avoid when:**
- You need pixel-faithful fidelity or full preservation of source styling (complex DOCX↔DOCX, InDesign-grade layout) — the AST will drop what it can't model.
- You only convert Markdown to HTML and want zero dependencies — a lightweight library (markdown-it, comrak) is smaller and embeddable.
- You need in-process conversion inside a non-Haskell app without shelling out — bindings exist but the CLI/subprocess model is the supported path.
- You require streaming conversion of documents too large to hold in memory.

## Alternatives

- jgm/commonmark-hs — same author; when you only need a strict CommonMark/GFM Markdown parser rather than universal conversion.
- markdown-it/markdown-it — use when you need an embeddable, extensible Markdown-to-HTML renderer in JavaScript with a plugin ecosystem.
- kivikakk/comrak — use when you want a fast Rust CommonMark/GFM library to embed rather than a CLI covering many formats.
- asciidoctor/asciidoctor — use when your source of truth is AsciiDoc and you want its native toolchain instead of pandoc's lossy AsciiDoc support.
- quarto-dev/quarto-cli — use when you want an opinionated scientific-publishing layer (it wraps pandoc) rather than the raw converter.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2008 | First 1.0 release of the converter[^1]. |
| 2.0 | 2017-10 | Major release; introduced Lua filters and internal restructuring[^4]. |
| 3.0 | 2023-01 | Split into subpackages; API and default changes across readers/writers[^5]. |
| — | 2026-07-14 | Actively maintained; latest push per GitHub API. |

## References

[^1]: Pandoc project — John MacFarlane, maintainer; project history on the website and repository. https://pandoc.org/
[^2]: Pandoc README — "The universal markup converter," modular readers/writers design and the note that the intermediate representation is less expressive than many supported formats. https://github.com/jgm/pandoc/blob/main/README.md
[^3]: Pandoc User's Guide — creating PDFs and `--pdf-engine` options. https://pandoc.org/MANUAL.html#creating-a-pdf
[^4]: Pandoc Lua filters documentation. https://pandoc.org/lua-filters.html
[^5]: Pandoc releases and changelog. https://github.com/jgm/pandoc/releases

## Tags

haskell, document-conversion, markdown, latex, pandoc, publishing, cli, markup, ast, commonmark, citations, static-site
