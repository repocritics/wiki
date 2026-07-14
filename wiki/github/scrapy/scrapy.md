# scrapy/scrapy

> A batteries-included web crawling framework for Python, built on the Twisted async networking engine.

[GitHub repo](https://github.com/scrapy/scrapy) ·
[Official website](https://scrapy.org) ·
[License: BSD-3-Clause](https://github.com/scrapy/scrapy/blob/master/LICENSE)

## Overview

Scrapy is a framework for extracting structured data from websites at scale.
It is not a scraping *library* you call from your own loop — it is an
application framework with its own control flow, into which you plug spiders,
pipelines, and middlewares. First public-released in 2010[^1] and maintained
by Zyte (formerly Scrapinghub) alongside a large contributor base[^2], it is
one of the oldest continuously developed projects in the Python data-collection
space and remains actively maintained, with commits landing the same week as
this writing.

The defining design decision — and the source of most of its tradeoffs — is
that Scrapy is built on **Twisted**, an event-driven async networking engine
that predates `asyncio` by years[^3]. This gives Scrapy genuine concurrency
(thousands of in-flight requests in a single process, single thread) without
the caller writing any async code. The cost is that the whole framework runs
inside a Twisted reactor with its own conventions, which is where newcomers and
integrators most often get cut. Scrapy has since added an `asyncio` reactor and
lets you `await` coroutines in callbacks, but the Twisted substrate is still
load-bearing and visible.

Scrapy is aimed at the "many pages, structured output, be a good citizen"
problem: crawl a site or many sites, follow links by rules, extract fields with
CSS/XPath selectors, and stream results to files, databases, or feeds — with
throttling, retries, dedup, and robots.txt handling built in. It is
deliberately not a browser: it fetches and parses HTTP responses and does not
execute JavaScript.

## Getting Started

```bash
pip install scrapy          # requires Python 3.10+
scrapy startproject books   # scaffolds a project layout
```

```python
# books/spiders/quotes.py — run with: scrapy crawl quotes -o out.jsonl
import scrapy

class QuotesSpider(scrapy.Spider):
    name = "quotes"
    start_urls = ["https://quotes.toscrape.com/"]

    def parse(self, response):
        for q in response.css("div.quote"):
            yield {
                "text": q.css("span.text::text").get(),
                "author": q.css("small.author::text").get(),
            }
        next_page = response.css("li.next a::attr(href)").get()
        if next_page:
            yield response.follow(next_page, self.parse)
```

`yield`-ing a dict emits an item; `yield`-ing (or `follow`-ing) a request
schedules another fetch. The reactor drives both. `scrapy shell <url>` drops
you into a REPL with a live `response` object for iterating on selectors.

## Architecture / How It Works

Scrapy is organized as a data-flow around a central **Engine** that coordinates
five main components[^4]:

- **Scheduler** — queues and dedups outgoing `Request` objects (fingerprint-based
  dedup by default).
- **Downloader** — performs the actual HTTP fetch via Twisted, honoring
  concurrency and delay settings.
- **Spiders** — your code. Parse callbacks receive a `Response` and yield items
  and/or more requests.
- **Item Pipeline** — an ordered chain that processes each scraped item
  (validation, dedup, DB writes, dropping).
- **Downloader / Spider Middlewares** — two hook layers that wrap requests and
  responses on the way in and out (cookies, retries, redirects, robots.txt,
  user-agent rotation, caching all ship as default middlewares).

Selectors are provided by **parsel** (a separate Zyte library) wrapping lxml,
so `response.css()` and `response.xpath()` share one engine. HTTP responses are
lazily parsed. The `CrawlSpider` subclass adds declarative link-following via
`Rule`/`LinkExtractor` for whole-site crawls without hand-written pagination.

Everything runs inside a single Twisted **reactor**. Because the reactor is a
process-global singleton that cannot be restarted, "run a crawl" is not an
ordinary function call — you use `CrawlerProcess` (owns the reactor) or
`CrawlerRunner` (you own it) to start crawls. This one fact drives most of the
"why can't I just call it in a loop / from Flask / twice" friction. Callbacks
must not block: any synchronous CPU-bound or blocking-IO work in a `parse`
method stalls every concurrent request, since it all shares one thread.

## Production Notes

**No JavaScript execution.** Scrapy sees the raw HTML the server returns. For
SPA / client-rendered sites you need `scrapy-playwright` or a headless-browser
sidecar; this is the single most common surprise for people scraping modern
sites.

**Reactor lifecycle is the top footgun.** The reactor is not restartable, so
running Scrapy inside a long-lived process (web request handler, Jupyter,
Airflow worker) requires care — `CrawlerRunner` + an externally managed reactor,
or running crawls as subprocesses. Mixing Twisted `Deferred`s and `asyncio` is
supported but has sharp edges; opt into the asyncio reactor explicitly and keep
your awaited code well-behaved.

**Politeness and throttling.** `CONCURRENT_REQUESTS`, `DOWNLOAD_DELAY`, and
`AUTOTHROTTLE_ENABLED` govern load. New projects default to obeying robots.txt
(`ROBOTSTXT_OBEY`), which will silently skip disallowed URLs — a frequent "why
is nothing being scraped" cause. Set concurrency per-domain, not just globally.

**Resumable jobs.** Large crawls hold their pending-request queue in memory by
default. Set `JOBDIR` to persist the scheduler/dupefilter state to disk so an
interrupted crawl can resume instead of restarting.

**Distribution is not built in.** Scrapy is single-process by design. Horizontal
scaling means an external shared queue: `scrapy-redis` (shared Redis scheduler)
or Frontera/scrapy-cluster are the common patterns. There is no native
multi-machine coordinator.

**Deployment.** `scrapyd` is the reference daemon for scheduling spider runs
over an HTTP API; Zyte Scrapy Cloud is the maintainer's hosted option. Both are
optional — many teams just run `scrapy crawl` under cron or a container.

**Upgrade pain.** Scrapy 2.0 dropped Python 2 and reworked internals; recent
minor releases have steadily deprecated old request-fingerprinting and settings
APIs, so long-lived projects accumulate deprecation warnings that occasionally
become hard breaks across minor versions. Pin the version.

## When to Use / When Not

**Use when:**
- You are crawling many pages / many sites and want dedup, retries, throttling,
  and feed export without building the plumbing.
- Target sites are server-rendered HTML (or you pair with a browser layer).
- You want extraction, crawl logic, and output pipelines as declarative,
  testable components rather than a script.

**Avoid when:**
- You need to scrape a handful of URLs — `requests` + `beautifulsoup4` is far
  less ceremony.
- The site is JS-heavy and you'd rather drive a real browser directly — reach
  for Playwright without the framework overhead.
- You need Scrapy embedded inside an existing async app; the reactor coupling
  fights `asyncio`/uvicorn-style stacks.

## Alternatives

- psf/requests + BeautifulSoup — the simple path for small, one-off scrapes; no framework, no concurrency, no crawl machinery.
- microsoft/playwright-python — use instead when the site requires a real browser and JavaScript execution rather than raw HTTP.
- scrapy/parsel — use when you only want Scrapy's CSS/XPath selector engine inside your own fetch loop.
- gocolly/colly — use when you want a Scrapy-shaped crawler but in Go, with goroutine concurrency instead of a reactor.
- apify/crawlee — use for a JS/TS-first (also Python) crawler with tight headless-browser integration and a managed-platform story.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010-02 | Repository created; open-sourced out of Mydeco/Insophia work[^1]. |
| 1.0 | 2015-06 | First stable 1.x; API consolidation.[^5] |
| 1.1 | 2016-05 | Python 3 support (alongside Python 2).[^5] |
| 2.0 | 2020-03 | Dropped Python 2; added `asyncio`/coroutine support in callbacks.[^5] |
| 2.x | 2020–2026 | Ongoing: asyncio reactor, new request fingerprinting, add-ons, steady deprecations. Python 3.10+ required as of current releases[^2]. |

## References

[^1]: scrapy/scrapy on GitHub — repository created 2010-02-22. https://github.com/scrapy/scrapy
[^2]: Scrapy README — "requires Python 3.10+ ... maintained by Zyte and many other contributors." https://github.com/scrapy/scrapy/blob/master/README.rst
[^3]: Twisted — event-driven networking engine underlying Scrapy. https://twisted.org/
[^4]: Scrapy docs, "Architecture overview." https://docs.scrapy.org/en/latest/topics/architecture.html
[^5]: Scrapy release notes / changelog. https://docs.scrapy.org/en/latest/news.html

## Tags

python, web-scraping, web-crawling, framework, twisted, data-extraction, html-parsing, xpath, css-selectors, async, spider
