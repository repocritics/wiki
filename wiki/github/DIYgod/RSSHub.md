# DIYgod/RSSHub

"Everything is RSSible" — a TypeScript service that scrapes hundreds of sites (Twitter, Weibo, Bilibili, Pixiv, Spotify, YouTube, etc.) and exposes them as RSS feeds.

## What it is

A community-built service that fronts sites without RSS feeds with a normalized RSS endpoint. Visit `https://rsshub.app/twitter/user/elonmusk` and get an RSS feed of @elonmusk's tweets. Each "route" is a hand-written scraper for a specific site / endpoint, contributed via PR. Operates the public `rsshub.app` instance and is self-hostable. AGPL-3.0 licensed.

## Key features

- 600+ "routes" — per-site, per-feature scrapers.
- Coverage focused on (but not limited to) Chinese-language sites: Weibo, Bilibili, Zhihu, V2EX, Lofter, plus international (Twitter, Instagram, TikTok, Pixiv, Spotify, YouTube).
- Self-hostable via Docker for higher rate limits or sites that block the public instance.
- Web-rendered + headless-browser scraping support for JS-rendered pages.
- AGPL-3.0 licensed.

## Tech stack

- TypeScript primary.
- Node.js runtime.
- Cheerio / Playwright for scraping.
- Docker-first self-host.

## When to reach for it

- You want RSS feeds for sites that don't ship them (TikTok, Instagram, Weibo, etc.).
- You're rebuilding your news / content pipeline around RSS sovereignty.
- You're operating a self-hosted feed aggregator (FreshRSS, Miniflux) and want RSSHub as the scraper layer.

## When *not* to reach for it

- You need vendor-stable feeds — sites change, scrapers break; the public RSSHub instance has variable uptime.
- You're allergic to AGPL.
- Site ToS prohibits scraping in your use case.

## Maturity signal

44k stars, 10k forks, AGPL-3.0, actively maintained. The 600+ route count reflects sustained community contribution. Open-issues count of 359 mostly tracks per-site breakage.

## Alternatives

- Site-native RSS where available — always prefer it.
- `feedbin` / `feedly` (commercial) — managed feed aggregators with built-in scrapers.
- Web-scraping libraries directly for one-off use.

## Tags

typescript, rss, scraping, nodejs, agpl, self-hosted, awesome-list, content-aggregation
