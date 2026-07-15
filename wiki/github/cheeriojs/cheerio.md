# cheeriojs/cheerio

> jQuery's core API reimplemented over a server-side HTML/XML parse tree — no browser, no layout, no JavaScript execution.

[GitHub repo](https://github.com/cheeriojs/cheerio) ·
[Official website](https://cheerio.js.org) ·
[License: MIT](https://github.com/cheeriojs/cheerio/blob/main/LICENSE)

## Overview

Cheerio parses an HTML or XML string into an in-memory node tree and exposes a
subset of jQuery's traversal and manipulation API over it. It was created by
Matthew Mueller in 2011 as a lighter alternative to running full jQuery on top of
jsdom, and is now maintained largely by Felix Böhm (fb55), who also maintains most
of the parsing stack underneath it[^1]. As of 2026 it is one of the most-depended-on
packages in the npm ecosystem, pulled in by nearly every Node-based scraping,
templating, and HTML-post-processing tool.

The single most important thing to understand about Cheerio is what it is *not*: it
is not a browser and not a DOM implementation. It does not run scripts, compute
styles, perform layout, fire events, or fetch subresources. `$('.price').text()`
reads the literal text that was in the HTML string you handed it. If that HTML came
from a client-rendered single-page app, the data you want was never in the markup and
Cheerio cannot help you — this is the recurring source of "Cheerio returns empty"
questions.

What you get in exchange for dropping all browser semantics is speed and simplicity:
Cheerio's node objects come from the lightweight `domhandler` model, not from a
spec DOM, so parsing and traversing large documents is cheap and predictable. The
jQuery-shaped API (`.find`, `.attr`, `.text`, `.each`, `.map`) is familiar to a huge
population of developers and maps directly onto scraping and transformation work.

## Getting Started

```bash
npm install cheerio
```

```js
import * as cheerio from 'cheerio';

const $ = cheerio.load('<ul id="fruits"><li class="apple">Apple</li><li class="pear">Pear</li></ul>');

$('.apple').text();               //=> 'Apple'
$('#fruits li').length;           //=> 2
$('li').first().addClass('sel');

$.html();
//=> <html><head></head><body><ul id="fruits">...</ul></body></html>
```

A typical scrape combines `fetch` (Cheerio does no I/O of its own) with `load`:

```js
const html = await fetch('https://example.com').then(r => r.text());
const $ = cheerio.load(html);
const titles = $('h2.title').map((_, el) => $(el).text().trim()).get();
```

## Architecture / How It Works

Cheerio is a thin API layer over a stack of small libraries, most by the same
maintainer:

1. **Parser** — `cheerio.load()` uses [parse5](https://github.com/inikulin/parse5)
   by default, a spec-compliant HTML5 parser that reproduces browser error-recovery
   for malformed markup. Passing `{ xmlMode: true }` or using the `htmlparser2`
   options switches to [htmlparser2](https://github.com/fb55/htmlparser2), a faster,
   forgiving, streaming tokenizer[^2].
2. **DOM model** — parsed output is a `domhandler` tree of plain node objects
   (`tagName`, `children`, `parent`, `attribs`), not W3C DOM nodes. This is why only
   a handful of node properties exist and why there is no `getBoundingClientRect`,
   `querySelector` on elements, computed style, or event system.
3. **Selector engine** — CSS selectors are resolved by `css-select` + `css-what`,
   not by jQuery's Sizzle. Coverage is broad (attribute, pseudo-class, combinators)
   but not identical to a browser; some jQuery extensions like `:has()` behave
   subtly differently and a few live-DOM pseudo-classes are meaningless here.
4. **Serialization** — `dom-serializer` turns the tree back into a string for
   `$.html()` / `.prop('outerHTML')`.

Because the DOM is just data, a Cheerio object is a wrapper around an array of these
nodes plus a reference to the shared tree. Mutations edit the tree in place; there is
no reflow or invalidation to worry about, but also no reactivity.

The choice of parser is the main architectural lever. parse5 matches what a browser
would build and is the safe default for scraping real-world web pages; htmlparser2 is
meaningfully faster and streams, which matters for very large documents or throughput-
bound pipelines, at the cost of stricter behavior on broken markup.

## Production Notes

**No JavaScript execution — the number-one footgun.** Cheerio sees only the HTML as
delivered. React/Vue/Angular apps that render client-side, infinite-scroll content,
and anything behind `fetch`/XHR will be absent. When the target is a live app you need
a real browser (Playwright/Puppeteer) to render first, then optionally hand the
resulting HTML to Cheerio.

**Parser selection affects both correctness and speed.** Default parse5 is slower but
reproduces browser quirks; htmlparser2 mode can be several times faster on large
inputs. Benchmark against *your* documents before switching — the difference is
workload-dependent, and htmlparser2 may parse malformed pages differently than a
browser would.

**Entity decoding and XML mode.** By default Cheerio decodes HTML entities. For feeds,
sitemaps, and SVG use `xmlMode: true` (or `cheerio.load(str, { xml: true })`), which
switches to case-sensitive tags, self-closing elements, and htmlparser2. Loading XML
with the HTML defaults silently lowercases tag names and mangles namespaced elements.

**The 1.0 API cleanup is a real migration.** For roughly eight years the shipped
version on npm was a `1.0.0-rc.x` prerelease; stable **1.0.0** landed in 2024 as an
ESM-first, TypeScript rewrite[^3]. It removed the callable default export in favor of
`cheerio.load()`, tightened types, and changed some option shapes. Code and tutorials
written against the long-lived RCs may not paste cleanly onto 1.0.

**Memory on large batches.** Each `load()` builds a full tree and keeps it alive as
long as the `$` reference exists. Scraping thousands of pages in a loop without
letting `$` go out of scope holds every tree in memory. Scope the object per iteration
and let GC reclaim it.

**Not a sanitizer.** Cheerio will happily round-trip `<script>` and event-handler
attributes. It is a manipulation library, not an XSS filter — sanitize separately
(e.g. DOMPurify, `sanitize-html`) if output is rendered in a browser.

## When to Use / When Not

**Use when:**
- You have server-rendered HTML/XML (a fetched page, an email template, a feed) and
  want to query or rewrite it with familiar jQuery-style selectors.
- You need fast, dependency-light HTML parsing in Node without spinning up a browser.
- You're post-processing build output, Markdown-rendered HTML, or templating fragments.

**Avoid when:**
- The content is rendered client-side or depends on JS — use a headless browser.
- You need real DOM APIs, `MutationObserver`, layout, or to run page scripts — use jsdom.
- You only need to stream-extract a few fields from huge documents — a raw SAX-style
  htmlparser2 handler avoids building the whole tree.

## Alternatives

- puppeteer/puppeteer — use instead when the page renders content with JavaScript and you need a real Chromium to execute it first.
- microsoft/playwright — use instead when you need cross-browser headless automation, not just static HTML parsing.
- jsdom/jsdom — use instead when you need a spec-compliant DOM, script execution, or browser APIs in Node.
- fb55/htmlparser2 — use instead when you want low-level streaming parse events and don't need the jQuery API.
- taoqf/node-html-parser — use instead when you want a lighter, faster parser and can accept a smaller API surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2011 | Initial release by Matthew Mueller[^1]. |
| 0.22.0 | 2016 | Last of the 0.x line before the RC series. |
| 1.0.0-rc.1 | 2016 | Long-lived release-candidate series begins; the de facto version for years. |
| 1.0.0-rc.10 | 2021 | Selector engine and parser stack modernized during the RC period. |
| 1.0.0 | 2024 | Stable release: ESM-first, TypeScript rewrite, `load()`-centered API[^3]. |

## References

[^1]: Cheerio project home and history. https://cheerio.js.org
[^2]: htmlparser2 — forgiving streaming HTML/XML parser used as Cheerio's optional backend. https://github.com/fb55/htmlparser2
[^3]: Cheerio releases (1.0.0 stable and the preceding `1.0.0-rc` series). https://github.com/cheeriojs/cheerio/releases

## Tags

javascript, typescript, html-parser, web-scraping, jquery, dom, xml, nodejs, htmlparser2, parse5, server-side
