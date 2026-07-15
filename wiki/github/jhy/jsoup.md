# jhy/jsoup

> A Java library for parsing, traversing, cleaning, and manipulating real-world HTML — a WHATWG-conformant parser plus jQuery-style selectors and an XSS sanitizer.

[GitHub repo](https://github.com/jhy/jsoup) ·
[Official website](https://jsoup.org) ·
[License: MIT](https://github.com/jhy/jsoup/blob/master/LICENSE)

## Overview

jsoup is a Java HTML parser created and still primarily maintained by Jonathan Hedley, with a first public release in 2010[^1]. It parses HTML — including the malformed "tag soup" found on real websites — into a DOM that matches what modern browsers produce, because it implements the WHATWG HTML5 parsing algorithm[^2] rather than assuming well-formed input. On top of that DOM it layers a CSS/jQuery-style selector engine, an HTML sanitizer, and a small built-in HTTP client for fetching pages.

The defining characteristic is that jsoup is a *parser and DOM library, not a browser*. It reads the HTML you give it and builds a static tree; it does not execute JavaScript, apply CSS, or perform layout. For server-rendered pages and documents this is exactly what you want — it is fast, dependency-free, and has a compact API. For single-page apps whose content is assembled client-side, jsoup sees only the empty shell, and the correct answer is a headless browser instead.

jsoup's second identity is as a sanitizer: its `Cleaner` plus `Safelist` (renamed from `Whitelist` in 1.14.1[^3]) is one of the most widely used Java tools for scrubbing user-submitted HTML down to an allowed set of tags and attributes to prevent XSS. This puts it on a security-sensitive path, and its CVE history reflects that — the project has shipped several fixes for sanitizer-bypass and denial-of-service issues, so staying on a current version matters.

## Getting Started

Maven:

```xml
<dependency>
  <groupId>org.jsoup</groupId>
  <artifactId>jsoup</artifactId>
  <version>1.18.1</version>
</dependency>
```

Gradle:

```groovy
implementation 'org.jsoup:jsoup:1.18.1'
```

Parse a string, select with CSS, and sanitize:

```java
import org.jsoup.Jsoup;
import org.jsoup.nodes.*;
import org.jsoup.select.Elements;
import org.jsoup.safety.Safelist;

// Parse and query
Document doc = Jsoup.parse("<p>Hello <a href='/x'>world</a></p>");
Elements links = doc.select("a[href]");
String href = links.first().attr("abs:href");   // resolve against base URI
String text = doc.body().text();                  // "Hello world"

// Fetch over the network (blocking) and query
Document page = Jsoup.connect("https://example.com")
    .userAgent("Mozilla/5.0")
    .timeout(10_000)
    .get();

// Sanitize untrusted HTML against a safelist
String clean = Jsoup.clean("<script>alert(1)</script><b>ok</b>", Safelist.basic());
// -> "<b>ok</b>"
```

## Architecture / How It Works

jsoup is a single dependency-free jar organized around a few subsystems:

1. **Parser** (`org.jsoup.parser`) — a `Tokeniser` driven by a `CharacterReader` feeds a tree builder. `HtmlTreeBuilder` implements the WHATWG tokenization and tree-construction state machine, including the "adoption agency" recovery for misnested inline tags and the insertion-mode rules that let it recover from broken markup the way a browser does. A separate `XmlTreeBuilder` handles XML with lenient, well-formedness-agnostic parsing.
2. **DOM** (`org.jsoup.nodes`) — `Node` is the base type; `Element`, `TextNode`, `Comment`, `DataNode` extend it, and `Document` extends `Element`. Nodes carry attributes, ownership, and sibling/parent links. The tree is fully materialized in memory — jsoup is not a streaming (SAX-style) parser.
3. **Select** (`org.jsoup.select`) — a `QueryParser` compiles a CSS selector string into a tree of `Evaluator` objects, which is then run over the element tree. It supports most of CSS3 plus jsoup extensions such as `:contains(text)`, `:matches(regex)`, and `:has(selector)`. `selectXpath()` is also available, implemented by bridging to a W3C DOM.
4. **Safety** (`org.jsoup.safety`) — `Cleaner` walks a parsed fragment and keeps only the tags/attributes permitted by a `Safelist`, discarding everything else. This is deliberately allow-list based, not deny-list based.
5. **Connection** (`org.jsoup.Connection` / `HttpConnection`) — a thin blocking HTTP client historically built on `java.net.HttpURLConnection`, handling gzip, redirects, basic cookies, and multipart form posts.

The `W3CDom` helper converts a jsoup `Document` into a standard `org.w3c.dom.Document`, which is what enables `javax.xml.xpath` querying. Because jsoup's lenient parse can produce structures that a strict W3C DOM rejects, this conversion is the lossy seam in the library and the place XPath users hit surprises.

## Production Notes

**It does not run JavaScript.** This is the single most common misunderstanding. If a site renders its content client-side (React/Vue/Angular SPAs, infinite-scroll feeds), `Jsoup.connect(url).get()` returns the pre-hydration HTML and your selectors match nothing. The fix is a real browser — Playwright, Selenium, or HtmlUnit — to render first, then optionally hand the resulting HTML to jsoup for extraction.

**The built-in HTTP client is a convenience, not infrastructure.** `Jsoup.connect()` is single-request, blocking, and has no connection pooling or real async. For high-throughput crawling, use a dedicated client (`java.net.http.HttpClient`, OkHttp) with its own concurrency and retry handling, and call `Jsoup.parse(html, baseUri)` on the response body. This also decouples you from jsoup's networking behavior across upgrades.

**`maxBodySize` defaults to ~2 MB.** Responses larger than that are silently truncated, which can lop off the part of the page you wanted. Set `.maxBodySize(0)` to disable the cap when you know you need the whole document.

**Character-encoding detection can misfire.** jsoup infers charset from the HTTP header, a BOM, and `<meta charset>`. Pages that lie about their encoding or declare it late produce mojibake; pass the correct charset explicitly to `Jsoup.parse(InputStream, charsetName, baseUri)` when you know it.

**The DOM is not thread-safe for concurrent mutation.** Parsing separate documents on separate threads is fine, but a single `Document`/`Element` must not be mutated from multiple threads. There is no internal locking.

**Sanitizer scope is body fragments.** `Cleaner`/`Safelist` is designed to clean HTML *content*, not arbitrary attribute or URL contexts, and its safety depends on you choosing an appropriate safelist and keeping the library current. Several past CVEs (a DoS via crafted input fixed in the 1.14.x line, and a sanitizer bypass fixed in 1.15.3[^4]) landed exactly here. Treat a jsoup upgrade as a security update, not just a feature bump.

**Android needs desugaring.** On Android, enable core library desugaring with NIO support so the Java 8+ APIs jsoup uses are available on older API levels[^1].

**Memory, not streaming.** Because the full DOM is built in memory, very large documents are expensive. There is no incremental/event-based mode for bounded-memory processing of huge inputs.

## When to Use / When Not

**Use when:**
- You are parsing or scraping server-rendered HTML/XML and want browser-equivalent tree building.
- You want CSS-selector or DOM-traversal extraction with a compact, dependency-free API.
- You need to sanitize user-submitted HTML against an allow-list to prevent XSS.
- You want to programmatically edit HTML (rewrite links, add attributes, tidy output).

**Avoid when:**
- The content is rendered by client-side JavaScript — you need a headless browser.
- You need a high-concurrency crawler — use a real HTTP client and hand bodies to jsoup.
- You must process documents too large to hold fully in memory — jsoup has no streaming mode.
- You need a strict, conformance-checking HTML5 parser with SAX events rather than a lenient DOM.

## Alternatives

- HtmlUnit/htmlunit — headless Java browser that executes JavaScript; use it when the target page renders content client-side and jsoup sees an empty shell.
- validator/htmlparser — the WHATWG HTML5 parser behind the Nu validator; use it when you want strict conformance and SAX-style streaming instead of a lenient in-memory DOM.
- microsoft/playwright — full browser automation; use it when you must run JS, log in, or interact with the page before extracting.
- cheeriojs/cheerio — the Node.js equivalent (jQuery-like server-side HTML); use it when your stack is JavaScript rather than the JVM.
- apache/tika — content and text extraction across many document formats; use it when you are pulling text out of mixed PDFs/Office/HTML, not scraping structured HTML.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010 | First public release by Jonathan Hedley[^1]. |
| 1.14.1 | 2021-07 | `Whitelist` renamed to `Safelist`; XPath support added[^3]. |
| 1.15.3 | 2022-09 | Security fix for a sanitizer/XSS bypass (CVE-2022-36033)[^4]. |
| 1.16.1 | 2023 | Parser and performance improvements; ongoing WHATWG alignment. |
| 1.18.1 | 2024 | Continued maintenance release on the stable 1.x line. |

## References

[^1]: jsoup README and project site — Jonathan Hedley, jsoup: the Java HTML parser. https://jsoup.org
[^2]: WHATWG HTML Standard — the parsing model jsoup implements. https://html.spec.whatwg.org/multipage/
[^3]: jsoup 1.14.1 release notes — `Whitelist` → `Safelist` rename and XPath support. https://jsoup.org/news/release-1.14.1
[^4]: CVE-2022-36033 — jsoup sanitizer bypass, fixed in 1.15.3. https://github.com/jhy/jsoup/security/advisories/GHSA-gp7f-rwcx-9369

## Tags

java, html-parser, web-scraping, css-selectors, dom, xss-sanitization, xpath, html5, jvm, data-extraction
