# gocolly/colly

> A callback-driven scraping framework for Go — fast static-HTML crawling with per-domain rate limits, but no JavaScript rendering.

[GitHub repo](https://github.com/gocolly/colly) ·
[Official website](https://go-colly.org/) ·
[License: Apache-2.0](https://github.com/gocolly/colly/blob/master/LICENSE.txt)

## Overview

Colly is a scraping and crawling framework for Go, first tagged in 2018[^1]. Its model is a single `Collector` object onto which you register event callbacks — `OnRequest`, `OnHTML`, `OnResponse`, `OnError`, `OnScraped` — and then seed with one or more URLs. The Collector walks the request graph you generate (typically by calling `Visit` from inside an `OnHTML` handler) while enforcing concurrency, delay, and depth limits. It is one of the most-starred Go libraries in the scraping space and the default answer to "how do I write a crawler in Go."

The framework's defining constraint is that it operates purely on the HTTP response body. Colly fetches HTML over `net/http` and hands it to goquery/cascadia for CSS-selector matching[^2]; it does not run a browser, execute JavaScript, or build a DOM from client-side rendering. For server-rendered pages this makes it fast (the README claims >1k requests/sec on a single core[^1]) and memory-light. For single-page apps or any content assembled by client-side JS, Colly sees only the initial skeleton — the most common reason a Colly scraper "returns nothing" on a page that looks full in a browser.

The other tension is release cadence. Colly is widely deployed and its API is stable to the point of being frozen, but the project went roughly five years between tagged releases (v2.1.0 in 2020 to v2.2.0 in 2025)[^3]. It works and is maintained again, but treat it as mature-and-slow rather than fast-moving.

## Getting Started

```bash
go get github.com/gocolly/colly/v2
```

```go
package main

import (
	"fmt"

	"github.com/gocolly/colly/v2"
)

func main() {
	c := colly.NewCollector(
		colly.AllowedDomains("go-colly.org"),
		colly.MaxDepth(2),
	)

	// Called for every <a href> found in a fetched page.
	c.OnHTML("a[href]", func(e *colly.HTMLElement) {
		link := e.Attr("href")
		fmt.Println("link:", link)
		e.Request.Visit(link) // enqueue for crawling (dedup + depth-aware)
	})

	c.OnRequest(func(r *colly.Request) {
		fmt.Println("visiting", r.URL)
	})

	c.Visit("https://go-colly.org/")
}
```

Note the `/v2` in the import path: v2 is published as a distinct Go module, so the import path — not just the version selector — changes between v1 and v2. Copying v1 example code without editing the import is a frequent first-time error.

## Architecture / How It Works

A `Collector` holds the crawl configuration and the callback registry. The core loop is: dequeue a request → apply limit rules → issue the HTTP call → run response/HTML/XML callbacks → collect any new `Visit` calls those callbacks made → repeat. Deduplication is by URL (with an option to revisit), and depth is tracked per request so `MaxDepth` bounds the crawl.

Key subsystems, most in their own subpackages:

- **HTML/XML parsing** — `OnHTML(selector, cb)` uses goquery's CSS selector engine; `OnXML(xpath, cb)` uses XPath via the antchfx libraries. Handlers receive an `HTMLElement`/`XMLElement` scoped to the matched node.
- **Rate limiting** — `Limit(&colly.LimitRule{DomainGlob, Parallelism, Delay, RandomDelay})` sets per-domain concurrency and inter-request delay. Rules match by glob, so different hosts can have different budgets.
- **Async mode** — `colly.Async(true)` turns `Visit` non-blocking; you then call `c.Wait()` to block until the queue drains. Without async, `Visit` is synchronous and single-threaded.
- **Storage** — visited-URL and cookie state live behind a `storage.Storage` interface. Default is in-memory; `colly/storage` and companion modules provide Redis, SQLite3, and other backends, which is also how distributed crawling and resumable state work.
- **Queue** — `colly/queue` provides an explicit queue with a configurable backing store, used when you want more control than the built-in request buffer.
- **Proxy** — `colly/proxy` offers a round-robin proxy switcher; `SetProxyFunc` lets you rotate per request.
- **Extensions** — `colly/extensions` bundles small helpers like `RandomUserAgent` and `Referer`.
- **Debugger** — a pluggable `debug.Debugger` (e.g. `LogDebugger`) traces the request lifecycle.

`Clone()` produces a Collector that copies configuration but not callbacks, the standard pattern for running a second scraper (e.g. detail pages) with different limits from the listing crawler. Colly also reads configuration from environment variables (`COLLY_*`), which lets deployment tune behavior without code changes.

## Production Notes

- **No JavaScript execution.** This is the single biggest operational surprise. If target content is rendered client-side, Colly will not see it. The usual escape hatch is to drive a real browser with chromedp or playwright-go and feed the rendered HTML into goquery, at a large throughput and resource cost.
- **Robots.txt is honored by default.** Colly obeys `robots.txt` unless you set `IgnoreRobotsTxt`. Scrapers that "work locally but fetch nothing in production" are sometimes hitting a disallow rule.
- **Default concurrency is 1.** Out of the box Colly is synchronous and single-threaded; throughput requires `Async(true)` plus a `LimitRule` with `Parallelism > 1`, and then `c.Wait()`. Forgetting `Wait()` in async mode causes the program to exit before requests finish.
- **Callback-based control flow is stateful.** Because parsing happens inside callbacks, sharing data across pages means closures, `Request.Ctx`, or external state — there is no return-value pipeline. Non-trivial scrapers accumulate mutable shared state that is easy to get wrong under concurrency; guard shared maps/slices with a mutex when `Parallelism > 1`.
- **Release gap.** Between v2.1.0 (2020) and v2.2.0 (2025) there were no tagged releases, so much production usage pinned to `master` or to specific commits to get fixes[^3]. Two 2025 releases (v2.2.0, v2.3.0) resumed the tag stream; audit your pin.
- **Character encoding.** Colly auto-detects and converts non-UTF-8 responses, which is usually helpful but can mangle pages that declare the wrong charset; disable with `DetectCharset` handling if you see garbled text.

## When to Use / When Not

**Use when:**
- The target sites are server-rendered HTML and you want high throughput with a small footprint.
- You are already in Go and want per-domain rate limiting, dedup, depth control, and pluggable storage without assembling them yourself.
- You need resumable or distributed crawls backed by Redis/SQLite.

**Avoid when:**
- Content depends on client-side JavaScript — use a headless browser instead.
- You prefer a linear, return-value scraping style over registering callbacks.
- You need a maintained framework with frequent releases and a large plugin ecosystem — Python's Scrapy is far deeper on both.

## Alternatives

- scrapy/scrapy — mature Python crawling framework with middleware, pipelines, and a large ecosystem; use when you want depth and plugins over raw Go throughput.
- chromedp/chromedp — drive headless Chrome from Go; use when the target requires JavaScript rendering that Colly cannot do.
- PuerkitoBio/goquery — jQuery-style HTML parsing without the crawler; use when you already have the HTML and only need extraction.
- geziyor/geziyor — Go crawler with built-in JS rendering and metrics; use when you want a Colly-like API but need rendered pages.
- go-rod/rod — higher-level Go browser automation; use when chromedp feels too low-level for a JS-heavy scrape.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017-09-29 | Repository created[^4]. |
| v1.0.0 | 2018-05-14 | First tagged release[^1]. |
| v1.1.0 | 2018-08-28 | v1 iteration. |
| v1.2.0 | 2019-02-13 | Final v1 line. |
| v2.0.0 | 2019-11-28 | `/v2` module path; import path change[^3]. |
| v2.1.0 | 2020-06-08 | Last release before a multi-year pause. |
| v2.2.0 | 2025-03-27 | Resumes releases after ~5 years[^3]. |
| v2.3.0 | 2025-12-04 | Latest tagged release. |

## References

[^1]: Colly README — features and >1k req/sec claim. https://github.com/gocolly/colly/blob/master/README.md
[^2]: goquery — CSS-selector HTML parsing used by Colly's `OnHTML`. https://github.com/PuerkitoBio/goquery
[^3]: Colly release tags and dates (GitHub Releases API), v2.0.0 through v2.3.0. https://github.com/gocolly/colly/releases
[^4]: GitHub repository metadata for gocolly/colly (created 2017-09-29, Apache-2.0). https://github.com/gocolly/colly

## Tags

go, golang, web-scraping, crawler, scraper, spider, framework, html-parsing, goquery, rate-limiting
