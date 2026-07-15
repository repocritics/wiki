# asciidoctor/asciidoctor

> The reference Ruby implementation of the AsciiDoc language: parse AsciiDoc into a document model, convert it to HTML 5, DocBook 5, or man pages.

[GitHub repo](https://github.com/asciidoctor/asciidoctor) ·
[Official website](https://asciidoctor.org) ·
[License: MIT](https://github.com/asciidoctor/asciidoctor/blob/main/LICENSE)

## Overview

Asciidoctor is a text processor and publishing toolchain that reads AsciiDoc — a plain-text markup language with far richer semantics than Markdown — and converts it to HTML 5, DocBook 5, and man pages out of the box, with PDF and EPUB 3 available through separate gems. It was started in 2012 by Ryan Waldron as a Ruby port of the original Python `asciidoc.py`, and is now led by Dan Allen and Sarah White[^1]. It has become the de facto AsciiDoc implementation: GitHub and GitLab render `.adoc` files through it, and the broader Antora/AsciidoctorJ/Asciidoctor.js ecosystem all descend from this codebase.

The defining tension is language-versus-processor. For most of its life there was no formal AsciiDoc specification — the language was effectively "whatever Asciidoctor does." A separate specification effort under the Eclipse Foundation is underway, but as of the 2.x line the parser here remains the ground truth. This makes Asciidoctor authoritative but also means edge-case behavior is defined by implementation, not by a written grammar. A ground-up rewrite (project "Asciidoctor 3" / a new spec-aligned parser) has been discussed for years but the shipping line remains 2.x.

The second tension is Ruby-versus-reach. The core is Ruby, yet most consumers are not Ruby users: Java shops run it via AsciidoctorJ (JRuby-packaged), JavaScript users via Asciidoctor.js (transpiled with Opal), and CI pipelines via the Gradle/Maven plugins. Those downstream packages inherit this repo's behavior but lag its releases and add their own translation-layer quirks.

## Getting Started

```bash
gem install asciidoctor
# or via OS packages: brew install asciidoctor / apt-get install asciidoctor
asciidoctor README.adoc          # -> README.html in the same directory
```

Ruby API — note the safe-mode footgun (the API defaults to `:secure`, which disables `include::`):

```ruby
require 'asciidoctor'

# Convert a file to a standalone HTML document
Asciidoctor.convert_file 'README.adoc', to_file: true, safe: :safe

# Parse to a document model, then convert — lets you inspect the AST
doc  = Asciidoctor.load_file 'README.adoc', safe: :safe
puts doc.doctitle
html = doc.convert
```

## Architecture / How It Works

The pipeline is three stages: **read → parse → convert**. The reader normalizes source lines and resolves `include::` directives and preprocessor conditionals (`ifdef`/`ifeval`) up front, before parsing. The parser is a hand-written, largely single-pass line-oriented parser (not a formal grammar or parser generator) that builds a tree of `AbstractNode` objects — `Document`, `Section`, `Block`, `List`, `Table`, `Inline`. Inline markup (bold, links, cross-references, attribute substitutions) is handled by a separate substitution pass over each block's text, driven by regular expressions.

Conversion is decoupled from parsing via the **converter** interface. A converter receives nodes and emits strings; the built-in HTML5, DocBook5, and manpage converters are ordinary Ruby classes. You can register your own converter, or override output piecemeal using the Tilt-backed template converter (ERB, Haml, Slim templates per node type). This clean parse/convert split is why the same document model can target wildly different backends and why PDF/EPUB live in separate gems that only supply a converter.

Extensions hook the pipeline at defined points: preprocessors, block/block-macro/inline-macro processors, tree processors, postprocessors, and include processors. Asciidoctor Diagram, for instance, is a block-macro extension that shells out to PlantUML/Graphviz. Extensions are registered globally in a per-process registry, which is convenient but means two documents converted in the same process share extension state unless you scope registration carefully.

Document attributes (`:name: value`) are the configuration substrate for everything: they gate conditional includes, parameterize substitutions, and toggle backend behavior. Attribute resolution has precedence rules (API/CLI overrides beat document-header values unless a value is soft-set with a trailing `@`), and getting this precedence wrong is a common source of "my attribute isn't taking effect" confusion.

The cross-platform story is a translation chain, not a reimplementation. **Asciidoctor.js** runs this exact Ruby source compiled to JavaScript by Opal; **AsciidoctorJ** runs it on the JVM via JRuby. Both are the same parser, which is the point — behavior parity — but it also means both carry a runtime bridge (Opal's Ruby-in-JS runtime, or JRuby startup cost) rather than idiomatic native code.

## Production Notes

- **Safe mode asymmetry is the top surprise.** The CLI defaults to `:unsafe`, but the Ruby API defaults to `:secure`, where `include::`, custom stylesheets, and several macros are disabled. Code that works from the shell silently drops includes when moved into a script. Set `safe: :server` (recommended) or `:safe` explicitly, and never run `:unsafe`/`:server` on untrusted input — `include::` can read arbitrary files.
- **Regex-based substitution has sharp edges.** Because inline formatting is regex-driven with a fixed substitution order, constructs like passthroughs (`+...+`, `pass:[]`), escaped markup, and attribute references interact in ways that are occasionally counterintuitive. When output looks wrong, the fix is usually understanding the substitution pipeline, not a parser bug.
- **Startup cost dominates for one-shot conversions.** Loading the gem and its regex tables costs real time; converting thousands of small files one-per-process is dramatically slower than reusing a single Ruby process or loading the document once. On the JVM, JRuby cold start makes short-lived AsciidoctorJ invocations expensive — long-running processes or the Gradle daemon amortize this.
- **Ecosystem versions drift.** AsciidoctorJ, Asciidoctor.js, and the build plugins pin specific core versions and release on their own cadence. A feature or fix in this repo may not reach the JVM or JS package for months. Pin versions across your toolchain rather than assuming parity.
- **PDF is a different renderer.** `asciidoctor-pdf` uses Prawn, not an HTML-to-PDF step, so CSS and HTML-backend theming do not apply; styling is a separate YAML theme system. Expect HTML output and PDF output to diverge.
- **Non-English Windows** can hit `Encoding::UndefinedConversionError`; the documented fix is `RUBYOPT="-E utf-8:utf-8"`. Asciidoctor assumes UTF-8 throughout.
- **Distro packages lag upstream.** The version in apt/dnf/apk repositories is frequently well behind the current gem; if you depend on recent syntax or fixes, install via `gem`/Bundler rather than the OS package, and pin the exact version across CI to avoid "works on my machine" divergence.

## When to Use / When Not

**Use when:**
- You need structured technical documentation — books, multi-file manuals, API docs — with includes, cross-references, admonitions, callouts, and tables that Markdown cannot express cleanly.
- You want one source converted to multiple formats (HTML, PDF, EPUB, DocBook, man page).
- You're building a docs site (Antora) or publishing on GitHub/GitLab, which render AsciiDoc natively via this engine.

**Avoid when:**
- Your content is short and flat and Markdown already suffices — AsciiDoc's power is overhead you won't use.
- You need a formally specified, portable markup with many independent conforming parsers today — AsciiDoc's behavior is still implementation-defined here.
- You want native performance without a Ruby/JRuby/Opal runtime in the loop, or you need to embed conversion in a latency-sensitive path with per-call process startup.

## Alternatives

- commonmark/cmark — use a Markdown processor instead when your documents are simple and portability across tools matters more than rich structure.
- jgm/pandoc — use when you need to convert *between* many document formats (DOCX, LaTeX, ODT, etc.); Pandoc has partial AsciiDoc support but is not the reference implementation.
- sphinx-doc/sphinx — use for Python-centric documentation with reStructuredText and a mature extension/theme ecosystem.
- rust-lang/mdBook — use for lightweight Markdown documentation sites when you don't need AsciiDoc semantics.
- asciidoctor/asciidoctorj — use this JVM packaging when your build and team live in Java/Gradle/Maven rather than Ruby.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2013-05 | First public gem; Ruby port of `asciidoc.py`[^1]. |
| 1.5.0 | 2014-08 | Rewritten substitution/extension model; API stabilization. |
| 1.5.6 | 2017 | Long-lived 1.5.x line widely deployed in CI/toolchains. |
| 2.0.0 | 2019-04 | Dropped Ruby < 2.3, removed deprecated APIs, converter cleanup[^2]. |
| 2.0.22 | 2026 | Current 2.x line; CRuby 2.7–3.4, JRuby 9.4–10, TruffleRuby[^3]. |

## References

[^1]: Asciidoctor README — history and authorship (project initiated 2012 by Ryan Waldron; led by Dan Allen and Sarah White). https://github.com/asciidoctor/asciidoctor#authors
[^2]: Asciidoctor 2.0.0 release notes. https://github.com/asciidoctor/asciidoctor/releases/tag/v2.0.0
[^3]: Requirements and supported Ruby implementations, Asciidoctor README. https://github.com/asciidoctor/asciidoctor#requirements

## Tags

ruby, asciidoc, documentation, static-site-generator, docbook, html, publishing, text-processor, markup, cli, rubygem
