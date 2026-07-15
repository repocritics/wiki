# thephpleague/commonmark

> Extensible PHP Markdown parser that tracks the CommonMark and GitHub-Flavored Markdown specs by mirroring the reference implementation.

[GitHub repo](https://github.com/thephpleague/commonmark) ·
[Official website](https://commonmark.thephpleague.com) ·
[License: BSD-3-Clause](https://github.com/thephpleague/commonmark/blob/2.8/LICENSE)

## Overview

`league/commonmark` is a PHP library that converts Markdown to HTML (and other
output via custom renderers). Written by Colin O'Dell and maintained under The
PHP League, its distinguishing design goal is fidelity to the
[CommonMark specification][^1]: the parser is a direct port of John MacFarlane's
`commonmark.js` reference implementation, and the maintainers explicitly avoid
refactors that would make it hard to follow upstream spec/reference changes[^2].
The deliberate tradeoff — it is not the fastest PHP Markdown parser, but it is the
most spec-correct and the most extensible.

Two ready-to-use converters ship in the box: `CommonMarkConverter` (plain
CommonMark) and `GithubFlavoredMarkdownConverter` (adds autolinks, tables,
strikethrough, task lists, and disallowed-raw-HTML filtering per the GFM spec).
Beyond those, almost everything is an extension over a shared `Environment`, so
teams can add or remove syntax, hook into parsing events, or emit non-HTML output
without patching the core. It is a common dependency in the PHP ecosystem —
Laravel, Drupal, Statamic, and many CMS/static-site tools pull it in for user
content rendering. The central adopter tension is security: CommonMark, by spec,
passes raw HTML through, so rendering untrusted input safely is opt-in
configuration — and getting it wrong is the most common misuse.

## Getting Started

Requires PHP 7.4+ with the `mbstring` extension. Install via Composer:

```bash
composer require league/commonmark
```

```php
use League\CommonMark\CommonMarkConverter;

$converter = new CommonMarkConverter([
    'html_input'         => 'strip',   // strip | escape | allow
    'allow_unsafe_links' => false,     // drop javascript:, data:, etc.
]);

echo $converter->convert('# Hello World!');
// <h1>Hello World!</h1>
```

For GitHub-Flavored Markdown, swap in the drop-in GFM converter:

```php
use League\CommonMark\GithubFlavoredMarkdownConverter;

$converter = new GithubFlavoredMarkdownConverter();
echo $converter->convert("~~strikethrough~~ and | tables |");
```

## Architecture / How It Works

Conversion is a pipeline over an **`Environment`** — a container that registers
block parsers, inline parsers, node renderers, extensions, event listeners, and a
merged configuration. The two bundled converters are thin wrappers that construct
an `Environment`, add `CommonMarkCoreExtension` (and, for GFM,
`GithubFlavoredMarkdownExtension`), and call the parser + renderer.

Parsing is two-phase, matching the reference implementation:

1. **Block parsing** — the `MarkdownParser` walks input line by line and builds
   the block structure (documents, lists, block quotes, code fences, headings)
   into an AST of `Node` objects.
2. **Inline parsing** — inline content inside blocks (emphasis, links, code
   spans, autolinks) is parsed in a second pass. Emphasis/link resolution uses
   the spec's delimiter-stack algorithm.

The resulting `Document` AST is handed to a renderer. `HtmlRenderer` is the
default; `XmlRenderer` emits the AST as XML for debugging or alternate pipelines.
Each node type maps to a `NodeRendererInterface`, so output is overridable per
element.

**Extensions** are the primary API surface: an extension's `register()` method
adds parsers/renderers to the `Environment`. Bundled extensions include Tables,
Strikethrough, Autolink, TaskList, Footnotes, Heading Permalinks, Table of
Contents, Attributes, External Links, Front Matter, Smart Punctuation, Mentions,
and Description Lists — most composable à la carte rather than as the whole GFM
bundle[^3]. An **event dispatcher** (e.g. `DocumentParsedEvent`) lets listeners
mutate the AST after parsing, which is how heading permalinks and external-link
rewriting are built. Configuration is validated against a schema, so unknown or
mistyped options fail loudly rather than being silently ignored.

## Production Notes

**Security is opt-in.** Per the CommonMark spec, raw HTML in the source is passed
through verbatim. For untrusted input you must set `html_input` to `strip` or
`escape` and `allow_unsafe_links` to `false`[^4]. Even then, the maintainers
recommend running output through a dedicated sanitizer such as HTML Purifier if
you allow any HTML — CommonMark is a parser, not an XSS filter.

**Denial-of-service via nesting.** Deeply nested or pathological input can blow up
parse time/memory. The `max_nesting_level` option caps block nesting depth; set
it when accepting arbitrary user input. Very long inputs with many delimiters can
also be expensive because of the delimiter-stack algorithm.

**Performance.** Spec-correctness costs speed: `league/commonmark` is markedly
slower than minimal parsers like Parsedown and slower than native C bindings.
Cache rendered HTML rather than re-parsing repeated content, and construct the
`Environment`/converter once and reuse it to avoid re-registering extensions per
request. Note also that only UTF-8 and ASCII input is supported — convert other
encodings to UTF-8 first, and ensure `mbstring` is enabled.

**Upgrade pain: 1.x → 2.0.** Version 2.0 was a substantial reorganization —
namespaces moved, the config/`Environment` API changed, `DocParser` became
`MarkdownParser`, and the extension and converter interfaces were reworked[^5].
Code written against 1.x does not run unmodified on 2.x; budget real migration
time. Within a major line, minor/patch releases follow SemVer but may still
change the emitted AST or HTML when the spec or a bug fix requires it, so pin and
test rendered output if byte-stable HTML matters. Support windows are short:
after a new minor ships the previous one gets critical/security fixes for at
least ~3 months, and a superseded major gets security fixes for ~6 months[^6].

## When to Use / When Not

**Use when:**
- You need output that actually conforms to the CommonMark and/or GFM specs.
- You want to extend or restrict Markdown syntax, or render to something other
  than default HTML, through a documented extension API.
- You are rendering user-generated content and want first-class, documented
  security configuration.

**Avoid when:**
- Raw throughput on trusted, simple Markdown dominates and you can accept a
  looser parser — a minimal single-file library will be faster.
- You are in a memory/latency-constrained context where pulling in the full
  extension architecture is overkill for a handful of formatting rules.
- You need a runtime with no `mbstring` or pre-7.4 PHP (unsupported).

## Alternatives

- erusev/parsedown — single-file, faster, widely embedded, but not CommonMark-spec
  compliant and effectively low-maintenance; use when speed and simplicity beat
  strict correctness.
- michelf/php-markdown — the classic PHP Markdown / MarkdownExtra port; use when
  you specifically want the original Markdown/MarkdownExtra dialect, not CommonMark.
- cebe/markdown — fast, extensible PHP parser with CommonMark and GFM flavors; use
  when you want a lighter dependency and can tolerate less complete spec coverage.
- jgm/commonmark.js — the JavaScript reference implementation; use in Node/browser
  contexts where PHP is not in play.
- commonmark/cmark — the canonical C reference implementation; use when you need
  maximum speed and can call native code instead of staying in pure PHP.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015 | Early releases as a PHP port of the CommonMark reference parser. |
| 1.0 | 2019-05 | First stable major; GFM extensions, extension architecture matured[^7]. |
| 2.0 | 2021-08 | Major rewrite of the config/Environment and extension APIs; PHP 7.4+ required[^5]. |
| 2.x | 2022–2026 | Ongoing minor releases; additional bundled extensions and spec updates. |

## References

[^1]: CommonMark specification. https://spec.commonmark.org/
[^2]: Contribution guidance on staying close to the reference implementation, from the project README. https://github.com/thephpleague/commonmark
[^3]: Bundled extensions overview. https://commonmark.thephpleague.com/extensions/overview
[^4]: Security documentation — handling untrusted input. https://commonmark.thephpleague.com/security/
[^5]: Upgrade / releases documentation. https://commonmark.thephpleague.com/releases
[^6]: Maintenance and support policy, project README. https://github.com/thephpleague/commonmark
[^7]: Packagist release history for league/commonmark. https://packagist.org/packages/league/commonmark

## Tags

php, markdown, commonmark, gfm, github-flavored-markdown, parser, html, text-processing, thephpleague, extensible
