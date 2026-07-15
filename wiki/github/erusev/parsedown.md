# erusev/parsedown

> A single-file, dependency-free Markdown parser for PHP — ubiquitous in the ecosystem, but effectively frozen since 2019.

[GitHub repo](https://github.com/erusev/parsedown) ·
[Official website](https://parsedown.org) ·
[License: MIT](https://github.com/erusev/parsedown/blob/master/LICENSE.txt)

## Overview

Parsedown is a Markdown-to-HTML parser written by Emanuil Rusev, first published in 2013[^1]. Its entire runtime is one class in one file (`Parsedown.php`) with no external dependencies, which is the reason it spread so widely: dropping one file into a project is cheaper than pulling a dependency tree. It is one of the most-installed PHP packages on Packagist, with well over a hundred million downloads, and ships inside Laravel, October CMS, Grav, Statamic, Kirby, Pico, phpDocumentor and many other projects[^2].

The design idea Parsedown popularized is "line-based" parsing: instead of a formal grammar, it reads input the way the author describes a human reading it — look at how each line starts to recognize blocks, then scan for special characters to recognize inline elements[^2]. This makes the parser fast and small, and made it easy to subclass, but it also means Parsedown is not a spec implementation. It passes most CommonMark tests but deliberately does not target full compliance, and its output diverges from CommonMark and GitHub Flavored Markdown on a long tail of edge cases.

The defining tension for anyone adopting Parsedown in 2026 is maintenance. The 1.x line has not had a stable release since 1.7.4 in December 2019, and the long-promised 2.0 rewrite has sat in perpetual pre-release for years[^3]. The repository still receives occasional commits (last push early 2026) and carries ~15k stars, but that activity has not translated into shipped releases. You are adopting a mature, battle-tested, effectively feature-frozen library — which is fine for stable Markdown, and a liability if you need spec fixes, new syntax, or timely security patches.

## Getting Started

```sh
composer require erusev/parsedown
```

Or download `Parsedown.php` from the latest release and `require` it directly — there is nothing else to install.

```php
<?php
require 'vendor/autoload.php';

$Parsedown = new Parsedown();

echo $Parsedown->text('Hello _Parsedown_!');
// <p>Hello <em>Parsedown</em>!</p>

// Inline-only parse (no wrapping block element):
echo $Parsedown->line('Hello _Parsedown_!');
// Hello <em>Parsedown</em>!

// Processing untrusted input — escape and sanitize markup:
$Parsedown->setSafeMode(true);
echo $Parsedown->text('[click](javascript:alert(1))');
```

Note the class is the global `Parsedown` — it is not namespaced (see Production Notes).

## Architecture / How It Works

Parsedown is a single class, historically ~2,000 lines, with no lexer/AST separation in the classic sense. Parsing runs in two conceptual passes over the "line-based" model:

1. **Block parsing.** `text()` normalizes line endings, splits input into lines, and walks them. The first non-whitespace character of a line is looked up in a `$BlockTypes` map (e.g. `#` → Header, `>` → Quote, `-`/`*`/`+` → List, `` ` `` / `~` → FencedCode). Each candidate block type has a `block<Type>` method that decides whether the line opens that block and how it continues. Blocks can be interrupted and continued line by line, which is what makes the approach streaming-friendly.
2. **Inline parsing.** Once block text is collected, `line()` scans it for inline markers registered in `$InlineTypes` (e.g. `_`/`*` → Emphasis, `` ` `` → Code, `[` → Link, `<` → Markup). It jumps between marker positions rather than tokenizing every character, which is where much of the speed comes from.

Extensibility is entirely through subclassing. Because block/inline handlers are `protected` methods keyed by marker character, you extend the language by subclassing `Parsedown` and overriding or adding handlers — this is exactly how the companion `erusev/parsedown-extra` adds Markdown Extra features (tables, footnotes, definition lists, abbreviations) by extending the base class[^4]. There is no plugin registry or middleware pipeline; the extension surface is OO inheritance, which couples every extension to Parsedown's internal method names and makes multiple independent extensions hard to compose.

Output is produced by building nested associative arrays describing elements (`name`, `handler`, `text`, `attributes`) and serializing them to HTML strings. Later 1.7.x versions moved toward an "Element" array shape to make sanitization and attribute handling more consistent.

## Production Notes

**Maintenance status is the headline risk.** The last tagged stable release is 1.7.4 (2019-12-30)[^3]. If you depend on Parsedown, treat its behavior as fixed: do not expect upstream fixes for CommonMark edge cases, new Markdown syntax, or PHP-version deprecations to arrive on a schedule. Several downstream projects have quietly migrated to `thephpleague/commonmark` for exactly this reason.

**Safe mode is a floor, not a wall.** `setSafeMode(true)` escapes raw HTML and sanitizes some scripting vectors introduced by Markdown syntax (e.g. `javascript:` link destinations), but the maintainer's own guidance is that it is not a substitute for a real HTML sanitizer[^2]. `setMarkupEscaped(true)` escapes markup but still permits unsafe URIs like `[x](javascript:alert(1))`. Parsedown has had multiple XSS advisories over its life, patched in point releases; because there are no new releases, you are relying on the current escaping logic being sufficient. For untrusted input, run output through HTML Purifier and deploy a Content-Security-Policy as defense in depth. Extensions bypass safe mode's guarantees entirely and must be audited separately.

**No namespace.** The class is the top-level global `Parsedown` (PSR-0-era design), not `Erusev\Parsedown\...`. This is harmless until it isn't: it prevents two incompatible versions from coexisting in one process and occasionally collides with other globals in large codebases. Autoloading still works via Composer's classmap.

**Not CommonMark/GFM compliant.** If your content is authored against GitHub's renderer, expect divergences — nested list tightness, HTML block boundaries, link reference edge cases, and some emphasis rules differ. Do not assume "renders on GitHub" implies "renders identically in Parsedown."

**Performance is genuinely good.** For single-pass, single-file simplicity it is fast and low-allocation, and instances are cheap to reuse across requests. This, plus zero dependencies, is why it remains attractive for CMSs and static-site tooling despite the freeze.

## When to Use / When Not

**Use when:**
- You need simple, fast Markdown rendering with zero dependencies and a trivial install.
- Your Markdown surface is stable and author-controlled (docs, CMS bodies you trust).
- You are maintaining an existing project already built on Parsedown and it works.
- You want a small, hackable base to subclass for a custom dialect.

**Avoid when:**
- You need CommonMark or GFM compliance, or expect ongoing spec fixes.
- You render untrusted input and want a maintained security posture with timely patches.
- You need composable extensions rather than single-inheritance overrides.
- You want a library with an active release cadence and a clear 2.x future.

## Alternatives

- thephpleague/commonmark — spec-compliant CommonMark + GFM, extensible via an event/environment system, actively maintained; the default modern choice when compliance or maintenance matters.
- michelf/php-markdown — the canonical classic-Markdown implementation plus Markdown Extra; use when you want reference "traditional" Markdown behavior.
- cebe/markdown — small, fast PHP parser with GFM and Markdown Extra flavors; use when you want a lighter, maintained alternative to league/commonmark.
- erusev/parsedown-extra — same author, adds Markdown Extra (tables, footnotes, definition lists) on top of Parsedown; inherits the same freeze and security caveats.
- knplabs/knp-markdown-bundle — Symfony integration layer (wraps a parser, not itself a parser); use when you just need Markdown wired into a Symfony app.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013 | Initial stable release; single-file line-based parser[^1]. |
| 1.6.0 | 2015 | Introduced `setSafeMode()` for untrusted input. |
| 1.7.0 | 2017 | Element-array output model, sanitization refinements. |
| 1.7.4 | 2019-12-30 | Last tagged stable release[^3]. |
| 2.0 | unreleased | Rewrite in perpetual pre-release; no stable tag as of 2026[^3]. |

## References

[^1]: Parsedown repository, created 2013-07-10. https://github.com/erusev/parsedown
[^2]: Parsedown README — features, "line based" approach, security guidance, and adopters (Laravel, October CMS, Grav, Statamic, phpDocumentor, and others). https://github.com/erusev/parsedown/blob/master/README.md
[^3]: Parsedown releases — latest stable tag is 1.7.4 (2019-12-30); the 2.0 line has not shipped a stable release. https://github.com/erusev/parsedown/releases
[^4]: erusev/parsedown-extra — Markdown Extra extension implemented by subclassing Parsedown. https://github.com/erusev/parsedown-extra

## Tags

php, markdown, markdown-parser, html, static-site, cms, security-xss, single-file, zero-dependency, unmaintained
