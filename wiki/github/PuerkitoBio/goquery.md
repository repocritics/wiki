# PuerkitoBio/goquery

> A jQuery-style API for querying and manipulating HTML in Go, built on the standard `net/html` parser and the cascadia CSS selector engine.

[GitHub repo](https://github.com/PuerkitoBio/goquery) ·
[Package reference](https://pkg.go.dev/github.com/PuerkitoBio/goquery) ·
[License: BSD-3-Clause](https://github.com/PuerkitoBio/goquery/blob/master/LICENSE)

## Overview

goquery gives Go a chainable, jQuery-like interface for traversing and manipulating HTML documents. It has been around since 2012[^1] and is the de facto default for HTML scraping and extraction in the Go ecosystem — most higher-level scraping frameworks (colly, ferret, Geziyor) either wrap it or borrow its selection model. The API deliberately mirrors jQuery method names (`Find`, `Each`, `Attr`, `Filter`, `Parent`, `Closest`) so that anyone who has written front-end JavaScript can be productive immediately.

The defining constraint is that goquery is a thin layer over two lower-level libraries it does not own: Go's `golang.org/x/net/html` parser and Andy Balholm's cascadia selector library[^2]. Because `net/html` returns a node tree rather than a full live DOM, goquery intentionally omits jQuery's stateful, rendering-dependent methods — there is no `height()`, `css()`, `offset()`, or layout computation, because there is no browser layout engine underneath. It is a parse-and-query tool, not a headless browser: it will not execute JavaScript, so content injected client-side is invisible to it.

A second constraint carried from `net/html`: the parser requires UTF-8 input. goquery does no charset detection or transcoding of its own; feeding it Latin-1, GBK, or Shift-JIS bytes yields mojibake, and it is the caller's responsibility to transcode first (typically via `golang.org/x/net/html/charset`)[^3]. The maintainer has declared the API stable and states it will not break[^1], which makes goquery a low-risk long-term dependency — the cost of that stability is that some rough jQuery-inherited edges (notably `Index`) are frozen in place.

## Getting Started

```bash
go get github.com/PuerkitoBio/goquery
```

```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/PuerkitoBio/goquery"
)

func main() {
	res, err := http.Get("https://example.com")
	if err != nil {
		log.Fatal(err)
	}
	defer res.Body.Close()

	// NewDocumentFromReader parses from any io.Reader; caller owns the request.
	doc, err := goquery.NewDocumentFromReader(res.Body)
	if err != nil {
		log.Fatal(err)
	}

	doc.Find("a").Each(func(i int, s *goquery.Selection) {
		href, exists := s.Attr("href")
		if exists {
			fmt.Printf("link %d: %s (%s)\n", i, s.Text(), href)
		}
	})
}
```

As of v1.12.0 (2026-03) goquery requires Go 1.25+; each recent minor release has bumped the floor in step with its dependencies[^1]. Development is tested against the latest two Go releases.

## Architecture / How It Works

goquery exposes two concrete types and one interface: `Document`, `Selection`, and `Matcher`. A `Document` is itself a `Selection` of one node (the root) — since v0.2.0 there is no separate `Document.Root`[^1]. Every traversal method returns a new immutable `*Selection` wrapping a slice of `*html.Node` pointers into the shared underlying tree. Because the nodes are shared, manipulation methods mutate the one parse tree that all selections point at, while the selection objects themselves are cheap value snapshots.

Selection happens through cascadia, which compiles a CSS selector string into a matcher and walks the node tree. This is the key coupling to understand: goquery's selector support **is** cascadia's, which follows `querySelectorAll` semantics, not jQuery's Sizzle engine. There is no contextual/scoped matching and no jQuery extensions like `:has()`-with-jQuery-semantics or `:contains()`[^4]. Invalid selector strings do not panic at the query site — since 2016 they compile to a matcher that matches nothing, so `doc.Find("~")` returns an empty selection rather than crashing[^1].

The API answers jQuery's overloaded, variadic-argument functions with statically typed method families instead. Where jQuery's `filter()` accepts a selector, a function, an element, or a jQuery object, goquery splits these into `Filter(string)`, `FilterFunction(func)`, `FilterNodes(...*html.Node)`, `FilterSelection(*Selection)`, and `FilterMatcher(Matcher)`. The `*Matcher` variants accept a pre-compiled cascadia selector, letting you compile once and reuse — this both avoids the panic risk of the string path and measurably reduces per-call overhead in hot loops[^1]. Utility operations that have no jQuery method equivalent (e.g. `NodeName`, `OuterHtml`, `Render`, the generic `Map`) are exposed as package-level functions taking a `*Selection`, to keep the method namespace reserved for jQuery-parity behavior.

## Production Notes

- **No JavaScript execution.** goquery sees only the HTML bytes you hand it. Single-page apps and content hydrated client-side will not be present. If a page renders its data via JS, you need a headless browser (chromedp, rod) upstream, then optionally feed the rendered HTML to goquery.
- **Encoding is your problem.** `net/html` assumes UTF-8. Non-UTF-8 pages must be transcoded before parsing (`charset.NewReader` from `golang.org/x/net/html`), or text extraction silently corrupts. There is no built-in `<meta charset>` sniffing.
- **`NewDocument(url)` and `NewDocumentFromResponse` are deprecated** (since v1.4.0, 2018) because they hid the HTTP client and gave you no control over timeouts, headers, redirects, or context cancellation. Do the `http.Get`/`http.Client.Do` yourself and use `NewDocumentFromReader`. This matters for any real scraper that needs a `context.Context`, custom User-Agent, or a request timeout[^1].
- **`Index()` is a known footgun** — the maintainer says so in the README[^5]. Its jQuery-inherited semantics (index within siblings vs. within the selection vs. relative to a selector argument) are easy to misread; read the godoc before relying on it.
- **The parser is lenient, so bad selectors fail quietly.** Because invalid selectors match nothing rather than erroring, a typo'd class name is indistinguishable from a page-structure change: both yield an empty selection. Assert on expected counts (`if s.Length() == 0`) rather than trusting silence.
- **Iterators, generics, and `Render` are recent.** Generic `Map` arrived in v1.9.0 (2024), `for..range` iteration via `EachIter` in v1.10.0, and `Render` (write a `Selection` to an `io.Writer`) in v1.8.0. Code targeting older goquery/Go versions won't have these[^1].
- **Concurrency.** A `*Selection` is a read-only snapshot and safe to read from multiple goroutines, but manipulation methods mutate the shared node tree — do not mutate one document concurrently. Parse one document per goroutine for parallel scraping.

## When to Use / When Not

**Use when:**
- You are scraping or extracting data from server-rendered HTML and want CSS selectors with familiar jQuery ergonomics.
- You need a stable, low-churn dependency with a frozen API and permissive license.
- You are post-processing HTML you already have (templating output, sanitization input, migration tooling).

**Avoid when:**
- The content is rendered client-side by JavaScript — goquery cannot see it; use a headless browser first.
- You need XPath rather than CSS selectors (use antchfx/htmlquery).
- You want a full crawling framework with rate limiting, queueing, and concurrency built in (use colly).
- You only need to pluck one value and want zero dependencies — hand-rolling with `net/html`'s tokenizer may be leaner.

## Alternatives

- gocolly/colly — full scraping framework (queueing, rate limits, caching); wraps goquery, use it when you need crawl orchestration, not just parsing.
- antchfx/htmlquery — use when you prefer XPath expressions over CSS selectors.
- andybalholm/cascadia — the selector engine under goquery; use directly when you want matches against raw `*html.Node` trees without the jQuery API.
- golang.org/x/net/html — the standard tokenizer/parser; use when you want zero abstraction and maximal control.
- andrewstuart/goq — declarative struct-tag unmarshaling built on goquery; use when you want to decode HTML straight into typed structs.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2012 | Initial release.[^1] |
| v0.2.0 | — | `Document` becomes a `Selection`; `Document.Root` removed; negative `Slice` indices.[^1] |
| v1.0.0 | 2016-07-27 | First tagged stable release.[^1] |
| v1.5.0 | 2018-11-15 | Go module support.[^1] |
| v1.8.0 | 2021-10-25 | `Render` a `Selection` to an `io.Writer`.[^1] |
| v1.9.0 | 2024-02-22 | Generic `Map` function; requires Go 1.18+.[^1] |
| v1.10.0 | 2024-09-06 | `EachIter` range-over-func iterator; requires Go 1.23+.[^1] |
| v1.11.0 | 2025-11-16 | Dependency bump; requires Go 1.24+.[^1] |
| v1.12.0 | 2026-03-15 | Dependency bump; requires Go 1.25+.[^1] |

## References

[^1]: goquery README and changelog. https://github.com/PuerkitoBio/goquery/blob/master/README.md
[^2]: cascadia — CSS selector library used by goquery. https://github.com/andybalholm/cascadia
[^3]: `golang.org/x/net/html/charset` — charset detection and transcoding for `net/html`. https://pkg.go.dev/golang.org/x/net/html/charset
[^4]: cascadia vs. jQuery selector semantics (querySelectorAll-style, no contextual matching). https://github.com/andybalholm/cascadia/issues/61
[^5]: goquery README, on the unintuitive `index()`. https://github.com/PuerkitoBio/goquery#readme

## Tags

go, html-parsing, web-scraping, css-selectors, jquery, dom, cascadia, net-html, data-extraction, library
