# xoofx/markdig

> The de facto CommonMark-compliant Markdown processor for .NET — fast, no-regex, and extensible down to the core parsers.

[GitHub repo](https://github.com/xoofx/markdig) ·
[Official website](https://xoofx.github.io/markdig/) ·
[License: BSD-2-Clause](https://github.com/xoofx/markdig/blob/main/license.txt)

## Overview

Markdig is a Markdown parser and HTML renderer for .NET, written by Alexandre Mutel (xoofx) and first published in 2016[^1]. It targets the [CommonMark](https://commonmark.org/) specification (0.31.2 as of recent releases) rather than a bespoke Markdown dialect, and ships 20+ optional extensions covering GitHub Flavored Markdown (pipe tables, task lists, autolinks, alerts), Pandoc-style constructs (grid tables, definition lists, footnotes), and editor-oriented features. It is the parser behind Visual Studio's Markdown Editor and a large share of .NET static-site and documentation tooling.

The defining design choice is a **fully pluggable pipeline**. Even core CommonMark block and inline parsing is exposed as swappable parser objects, so you can disable raw HTML parsing, change the character that starts a heading, or inject a custom block type without forking the library. This makes Markdig unusually good as an engine to build editors and custom dialects on top of — but it also means the "right" behavior depends entirely on how you configured the `MarkdownPipeline`, and defaults are permissive.

The second differentiator is **roundtrip parsing**: Markdig can retain trivia (whitespace, newlines, source offsets) so a document can be parsed, mutated, and re-rendered without spurious formatting changes[^2]. Combined with precise source-location tracking on every AST node, this is why it underpins interactive editors rather than only one-shot HTML conversion.

## Getting Started

```bash
dotnet add package Markdig
```

```csharp
using Markdig;

// Plain CommonMark — no extensions
string html = Markdown.ToHtml("This is *emphasis* and a [link](https://example.com)");

// Enable the advanced extension bundle (tables, footnotes, autolinks, etc.)
var pipeline = new MarkdownPipelineBuilder()
    .UseAdvancedExtensions()
    .Build();

string rich = Markdown.ToHtml("# Title\n\n| a | b |\n|---|---|\n| 1 | 2 |", pipeline);

// Parse to an AST instead of rendering
MarkdownDocument doc = Markdown.Parse("- [ ] todo\n- [x] done", pipeline);
```

`Markdig.Signed` is a separate NuGet package with strong-named assemblies for consumers that require it.

## Architecture / How It Works

Markdig is a two-phase parser feeding a visitor-based renderer:

1. **Block parsing** — the input is scanned line by line by an ordered set of `BlockParser` objects (headings, lists, fenced code, tables, etc.) that build a tree of `Block` nodes. Block structure is resolved first, before any inline content.
2. **Inline parsing** — each leaf block's text is re-scanned by `InlineParser` objects (emphasis, links, autolinks, math, emoji) to produce `Inline` nodes as children.
3. **Rendering** — a renderer walks the `MarkdownDocument` AST. `HtmlRenderer` is the default; `NormalizeRenderer` re-emits canonical Markdown, and `RoundtripRenderer` re-emits the original source using retained trivia.

Parsing avoids regular expressions entirely and works over `StringSlice` value types to keep allocations and GC pressure low — the performance claim in the README rests on this, not on unsafe tricks[^1]. An `IMarkdownExtension` mutates the `MarkdownPipelineBuilder` by registering or reordering parsers and renderer hooks; `UseAdvancedExtensions()` is just a convenience bundle of many such extensions. Because extensions register parsers into ordered lists, **extension order can affect output** when two extensions claim the same trigger character.

`MarkdownPipeline` is immutable once `Build()` is called. That immutability is what makes it safe to cache and share across threads, and it is the intended usage pattern.

## Production Notes

**Reuse the pipeline; never rebuild per call.** Constructing a `MarkdownPipelineBuilder`, wiring extensions, and calling `Build()` is comparatively expensive. A built `MarkdownPipeline` is immutable and thread-safe — build it once (a static field or DI singleton) and reuse it for every `ToHtml`/`Parse` call. Rebuilding per request is the most common performance mistake.

**Output is not sanitized — this is an XSS footgun.** By default Markdig passes raw inline and block HTML straight through, and does not filter `javascript:` or `data:` URIs in links. Rendering untrusted user Markdown to HTML and injecting it into a page is a stored-XSS vector. Mitigations: call `.DisableHtml()` on the builder to drop raw HTML, and run the rendered output through an HTML sanitizer (e.g. HtmlSanitizer/Ganss.Xss) before display. Do not rely on Markdig alone for safety.

**Version pinning and target frameworks.** From `0.20.0+`, Markdig requires `netstandard2.0`/`netstandard2.1` (and modern .NET); support for legacy .NET Framework 3.5/4.0 stops at `0.18.3`, which is the version to pin if you are stuck on those runtimes[^1]. Markdig is still `0.x` and has never declared a 1.0 — minor version bumps have occasionally changed extension behavior or output, so pin an exact version and diff rendered output on upgrade rather than floating.

**Roundtrip is opt-in and separate.** Standard rendering discards trivia; you must enable roundtrip trivia parsing and use `RoundtripRenderer` to get lossless edit-and-re-emit. Mixing a roundtrip-parsed document with the default `HtmlRenderer` works but the extra trivia nodes are simply ignored[^2].

**Custom extensions are a real surface.** Writing a custom block or inline parser means understanding the two-phase model and the parser-ordering rules; there is a developer guide, but the API is internals-heavy and sparsely tutorialized outside the official docs. Budget time for it.

## When to Use / When Not

**Use when:**
- You need CommonMark/GFM parsing in .NET and want spec compliance rather than a home-grown dialect.
- You are building a Markdown editor or tool that needs the AST, source positions, or lossless roundtrip.
- You need to add or disable syntax (custom containers, custom directives) without forking a parser.
- You want low allocation overhead for high-volume server-side rendering.

**Avoid when:**
- You are outside .NET — this is a .NET-only library; use a native parser for your stack.
- You render untrusted Markdown and are unwilling to add a separate sanitization step (the parser will not protect you).
- You need a frozen, 1.0-stable API with a strict semver contract — Markdig is still `0.x`.
- Your needs are trivial and fixed; a smaller single-purpose parser may be enough, though Markdig's plain path is already minimal.

## Alternatives

- Knagis/CommonMark.NET — the earlier .NET CommonMark parser; Markdig reuses some of its HTML-entity decoding. Now effectively superseded and unmaintained — only for legacy code already depending on it.
- commonmark/cmark — the C reference implementation; use when you need the canonical spec parser or bindings from a non-.NET runtime.
- yuin/goldmark — Go's extensible CommonMark parser with a similar pluggable-pipeline design; the natural choice on Go stacks.
- markdown-it/markdown-it — extensible JS/TypeScript parser for Node and browser environments.
- github/cmark-gfm — GitHub's CommonMark fork with GFM extensions in C; use when you must match GitHub's exact rendering.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016-05 | Initial public release; project created on GitHub[^1]. |
| — | 2016-06 | Author's "Implementing a Markdown Engine for .NET" writeup[^1]. |
| 0.18.3 | 2020 | Last release supporting .NET Framework 3.5 / 4.0. |
| 0.20.0 | 2020 | Baseline moved to netstandard2.0/2.1 and modern .NET[^1]. |
| 0.3x–0.4x | 2023–2026 | Tracks CommonMark spec 0.31.2; GitHub-style Alerts and CJK-friendly emphasis added. |

(Markdig has not released a 1.0; it versions continuously in the `0.x` range. Exact minor-version dates vary — pin and verify.)

## References

[^1]: Markdig README and Alexandre Mutel, "Implementing a Markdown Engine for .NET" — 2016. https://xoofx.github.io/blog/2016/06/13/implementing-a-markdown-processor-for-dotnet/
[^2]: Markdig Roundtrip documentation. https://github.com/xoofx/markdig/blob/main/src/Markdig/Roundtrip.md

## Tags

csharp, dotnet, markdown, commonmark, gfm, parser, markdown-to-html, ast, extensible, text-processing
