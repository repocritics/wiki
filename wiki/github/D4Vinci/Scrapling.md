# D4Vinci/Scrapling

An adaptive Python web scraping framework that spans single HTTP requests, stealth browser automation, and concurrent multi-session crawls, with a parser that relocates elements after a site's markup changes.

## What it is

Scrapling is a Python library and framework for extracting data from websites. It targets developers who need to scrape pages that change layout over time, sites protected by anti-bot systems, or workloads that grow from one-off requests into full crawls. The core idea is that a single library covers the whole range: a parser that tracks elements across markup changes, fetchers that impersonate browsers or drive real headless browsers, and a Scrapy-style spider framework for scaling out. It also ships an MCP server so AI assistants can drive scraping directly.

## Key features

- Adaptive element tracking: when a page's structure changes, elements can be relocated using similarity algorithms by passing `adaptive=True`, and selectors can be auto-saved for later re-matching.
- Multiple fetcher tiers: `Fetcher`/`AsyncFetcher` for HTTP with browser TLS-fingerprint impersonation and HTTP/3, `DynamicFetcher` for Playwright/Chrome browser automation, and `StealthyFetcher` for anti-bot bypass including Cloudflare Turnstile.
- Scrapy-like spiders: define `start_urls` and async `parse` callbacks with `Request`/`Response` objects, configurable concurrency and per-domain throttling, checkpoint-based pause/resume, and a streaming mode that yields items as they arrive.
- Session management and proxy rotation: persistent `FetcherSession`/`StealthySession`/`DynamicSession` classes, a built-in `ProxyRotator` with cyclic or custom strategies, domain/ad blocking (~3,500 known ad and tracker domains), and optional DNS-over-HTTPS to avoid DNS leaks behind proxies.
- Flexible selection and navigation: CSS and XPath selectors, BeautifulSoup-style `find_all`, text and regex search, `find_similar`, sibling/parent/child traversal, and auto-generation of CSS/XPath selectors.
- CLI and interactive shell: an IPython-based scraping shell plus `scrapling extract` commands that fetch a URL to `.txt`, `.md`, or `.html` without writing code.
- MCP server for AI assistants that extracts targeted content before passing it to the model, aimed at reducing token usage.

## Tech stack

- Language: Python, requiring Python 3.10 or higher; declares support through 3.13 (CPython).
- Core dependencies: `lxml>=6.1.1`, `cssselect>=1.4.0`, `orjson>=3.11.8`, `tld>=0.13.2`, `w3lib>=2.4.1`, `typing_extensions`.
- Optional `fetchers` extra: `curl_cffi>=0.15.0`, pinned `playwright==1.61.0` and `patchright==1.61.2`, `browserforge>=1.2.4`, `apify-fingerprint-datapoints>=0.13.0`, `msgspec>=0.21.1`, `anyio>=4.13.0`, `protego>=0.6.2`, and `click>=8.3.0`.
- Optional `ai` extra: `mcp>=1.27.0` and `markdownify>=1.2.0`; optional `shell` extra: `IPython>=8.37` and `markdownify`; `all` bundles both.
- Base install ships only the parser engine; fetchers and spiders require the extras plus `scrapling install` to download browsers.
- Packaging via setuptools with a `scrapling` console script; type-checked in CI with PyRight and MyPy; distributed as a Docker image bundling browsers.

## When to reach for it

- Scraping sites whose HTML structure shifts over time, where selectors otherwise break on each redesign.
- Targets behind anti-bot protection such as Cloudflare Turnstile that require browser or TLS-fingerprint spoofing.
- Projects that start as single requests but need to scale into concurrent, multi-session crawls with pause/resume.
- Crawls that mix cheap HTTP fetches with expensive stealth-browser fetches, routed by session within one spider.
- AI-assisted extraction workflows where an MCP server feeds pre-filtered content to an assistant.
- Quick one-off extraction from the terminal or an interactive shell without writing a full script.

## When *not* to reach for it

- Projects on Python older than 3.10, which is the minimum supported version.
- Static parsing needs where lxml, Parsel, or BeautifulSoup alone suffice, since browser and stealth features add heavier optional dependencies (pinned Playwright/Patchright, browser downloads) you would not otherwise pull in.
- Teams already invested in the Scrapy ecosystem and its middleware/plugin catalog, which Scrapling's spider API resembles but does not extend.
- Uses that conflict with local or international scraping and privacy laws; the project states it is for educational and research purposes and disclaims misuse.

## Maturity signal

Scrapling has drawn a large audience quickly, and the repository shows the activity to match: it was created in late 2024, was last pushed in mid-2026, and carries very few open issues alongside a permissive BSD-3-Clause license. The README reports 92% test coverage, full type hints checked by PyRight and MyPy on every change, and daily use over the past year, which points to active maintenance rather than a stalled project. Worth noting that `pyproject.toml` still marks the package "Development Status :: 4 - Beta" at version 0.4.11, so despite the reach the authors treat the API as pre-1.0 and potentially still changing.

## Alternatives

- Use Scrapy instead when you want a mature, plugin-rich crawling ecosystem and don't need built-in stealth or adaptive selectors.
- Use Playwright or Selenium directly when you need general browser automation beyond scraping and prefer controlling the browser stack yourself.
- Use BeautifulSoup or Parsel/lxml instead when you only need to parse static HTML and want a minimal dependency footprint.
- Use AutoScraper instead when you specifically want example-driven scraping and don't need fetchers, sessions, or a crawl framework.

## Notes

- The base `pip install scrapling` deliberately ships only the parser; importing from `scrapling.fetchers` or `scrapling.spiders` raises `ModuleNotFoundError` until the `fetchers` extra and `scrapling install` are run.
- Playwright and Patchright are pinned to exact versions (`==1.61.0` and `==1.61.2`), unlike the range-based constraints on the rest of the dependency set.
- The package version is intentionally static in `pyproject.toml` rather than dynamic, a choice the authors attribute to Docker build layer caching.
- Development mode caches responses to disk on first run and replays them, so you can iterate on `parse()` logic without re-hitting target servers.

## Tags

python, web-scraping, crawler, framework, library, browser-automation, anti-bot, playwright, cli, mcp, data-extraction, html-parsing
