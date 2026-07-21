# sansan0/TrendRadar

A self-hosted trending-news monitor that aggregates multiple platforms and RSS, filters by keyword or AI, and pushes summaries to chat channels.

## What it is

TrendRadar collects trending topics from multiple platforms plus RSS feeds, filters them down to what you care about, and pushes reports and AI analysis to your phone or team channels. It adds AI-based news filtering, translation, and analysis briefings, and can expose its data through an MCP server for conversational querying. It is aimed at people who want to cut through information overload with a lightweight, easy-to-deploy tool they run themselves.

## Key features

- Aggregates trending topics across multiple platforms and RSS/Atom feeds, with keyword and regex filtering.
- AI-based news filtering (natural-language interests), AI translation into other languages, and AI-generated analysis briefings including sentiment.
- Pushes to nine-plus channels: personal WeChat and WeCom, Feishu, DingTalk, Telegram, ntfy, Bark, Slack, email, and a generic Webhook, with per-channel formatting and batch splitting.
- An MCP server exposing analysis tools for clients such as Claude Desktop, Cursor, Cline, and Cherry Studio.
- Multiple deployment paths: GitHub Actions (zero-cost), GitHub Pages, Docker, Cloudflare Pages, and local runs.
- HTML reports with dark mode, wide-screen layout, live search, and export to image or Markdown.
- Storage via local SQLite or remote S3-compatible object storage such as Cloudflare R2.

## Tech stack

- Python `>=3.12`, packaged with hatchling; exposes `trendradar` and `trendradar-mcp` entry points.
- `fastmcp==2.12.5` for the MCP server and `litellm==1.82.6` as the unified LLM client.
- `feedparser==6.0.12` (RSS), `boto3==1.42.76` (S3 storage), `websockets`, `requests`, `pytz`, `PyYAML`, `json-repair`, `tenacity`.
- Official Docker images `wantcat/trendradar` and `wantcat/trendradar-mcp`; scheduling via GitHub Actions.

## When to reach for it

- Running low-cost or zero-cost trending-news monitoring with keyword filtering and phone push, without a server.
- Getting AI summaries and sentiment of news trends delivered into chat channels.
- Querying and analyzing collected news conversationally from an MCP-capable client.
- Self-hosting your own data in either SQLite or S3-compatible storage.

## When *not* to reach for it

- You need broad coverage of non-Chinese platforms; the upstream data comes from the newsnow API, which is China-centric.
- Your use case requires embedding in closed-source commercial software, which the GPL-3.0 license constrains.
- You want a hosted, no-setup product rather than a fork-and-run tool.
- Your environment cannot provide Python 3.12 or newer.

## Maturity signal

At roughly 61k stars, about 25k forks, active pushes within the last week of these facts, GPL-3.0, and version 6.10.0 with a separately versioned MCP server (v4.1.0), this is a very actively maintained project with a rapid release cadence. The unusually high fork count relative to stars reflects its fork-and-run deployment model rather than heavy source contribution.

## Alternatives

- **ourongxing/newsnow** — the upstream hot-list data source; use it directly when you only want the aggregated dashboards without filtering, AI, or push.
- **RSSHub** — use it when your need is turning many sources into RSS feeds rather than filtering and delivering summaries.
- **FreshRSS** — use it when you just want to self-host and read feeds without the AI analysis and multi-channel push.

## Notes

The very high fork count is explained by the deployment model: users fork the repository and run it via GitHub Actions on a schedule. The project relies on newsnow's public API by the author's permission, and the README asks Docker users to keep push frequency reasonable. The AI layer was migrated to LiteLLM (supporting 100+ providers) in v5.3.0, and the MCP server is maintained on its own version track and Docker image.

## Tags

python, news, rss, llm, mcp, monitoring, docker, automation, notifications, aggregator
