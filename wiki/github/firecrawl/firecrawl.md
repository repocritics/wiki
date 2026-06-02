# firecrawl/firecrawl

An API for crawling, scraping, and interacting with the web at scale — purpose-built for LLM agent pipelines that need "give me this page as markdown".

## What it is

A TypeScript service that turns the messy web into LLM-friendly inputs: pages → clean markdown, sites → structured data, search → ranked results. Designed for agentic pipelines where the bottleneck is often "I have a URL, I need its content as model context". Provides scraping, crawling, mapping, and search endpoints, all returning markdown / JSON / HTML in the shape an agent actually wants. Self-hostable (Docker, Kubernetes) or hosted at firecrawl.dev. AGPL-3.0 licensed — the network-service copyleft.

## Key features

- Scrape: single page → clean markdown / HTML / structured fields via LLM-extracted schema.
- Crawl: spider an entire site with depth + URL pattern controls.
- Map: discover all URLs on a site without fetching content.
- Search: query the web and return results in agent-ready shape.
- Anti-bot handling: handles common defenses (JavaScript-rendered pages, basic anti-scrape patterns).
- Markdown is the default output — strips ads, navigation, and boilerplate.
- Self-hostable for privacy or rate-limit reasons.
- AGPL-3.0 licensed.

## Tech stack

- TypeScript primary across the service.
- Node.js runtime; Playwright for JS-rendered page handling.
- Docker / Kubernetes deployment.
- Hosted offering at firecrawl.dev with API keys.

## When to reach for it

- You're building an LLM agent that needs to fetch + clean web content as model input.
- You want a single API surface for scrape / crawl / map / search rather than wiring four tools together.
- You need self-hostable web-scraping infrastructure for privacy or rate-limit reasons.

## When *not* to reach for it

- You're doing one-off web scraping — `requests + BeautifulSoup` or `playwright + readability` is lighter.
- You want a polished search-engine alternative — Firecrawl's search is agent-pipeline-shaped, not a Google replacement.
- You're allergic to AGPL — the copyleft pulls hosting services into source-release territory.

## Maturity signal

127k stars, 7.6k forks, AGPL-3.0, last push the day this page was generated. 2-year-old project (April 2024 origin) that became the canonical "web → agent input" service quickly. Active push cadence. The 370 open-issues count is moderate. The combination of "self-hostable + paid SaaS + AGPL" is the modern commercial-OSS pattern.

## Alternatives

- Direct `playwright` / `selenium` + `readability` — use for one-off scraping without a service abstraction.
- Bright Data, ScrapingBee, Apify — use for managed scraping with anti-bot handling at scale.
- `tavily-ai/tavily` — use when you specifically want agent-pipeline search rather than full scrape/crawl.
- `crawl4ai` — use when you want a Python-native LLM-friendly crawler.

## Notes

The AGPL license is intentional — it makes SaaS hosting Firecrawl-as-a-service legally meaningful, which protects the commercial offering. Anyone self-hosting for internal use is fine; reselling the service requires AGPL compliance. The "markdown by default" output choice fits how LLMs actually consume web content and is the reason this project surged ahead of generic scraping tools.

## Tags

typescript, web-scraping, crawler, agent, large-language-model, markdown, agpl, self-hosted, web-search, api
