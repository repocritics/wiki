# mvanhorn/last30days-skill

An AI agent skill that searches many social platforms, prediction markets, and the web in parallel, scores results by real engagement, and synthesizes them into one grounded, source-cited brief.

## What it is

`/last30days` is an agent skill that researches any topic — a person, company, product, or "X vs Y" comparison — by searching sources such as Reddit, X, YouTube, TikTok, Hacker News, Polymarket, GitHub, arXiv, and the web at once. Instead of ranking by SEO, it scores results by what people actually engaged with (upvotes, likes, views, market odds) and has an agent judge merge cross-source clusters into a single brief. It is aimed at people who need recent, community-sourced information that general web search misses — before a meeting, a sales call, a trip, or building something. It installs into Claude Code, Codex, Cursor, Copilot, Gemini CLI, Grok, Claude Desktop, OpenClaw, and other Agent Skills hosts.

## Key features

- Parallel search across 20+ sources scored by engagement signals, with same-story clusters merged across platforms.
- Zero-config start for Reddit (with comments), Hacker News, Polymarket, and GitHub; a first-run wizard unlocks more sources.
- Bring-your-own keys and browser sessions for gated platforms (X, YouTube, TikTok, Instagram, LinkedIn, Perplexity, and others).
- Modes beyond topic lookup: discovery ("what's trending"), single-pass comparisons, `--hiring-signals`, and ELI5.
- Output as markdown, self-contained HTML briefs (`--emit=html`), or versioned JSON (`--emit=json`); a subscribable library feed with Atom and offline library search.
- Trend monitoring via `--store` into SQLite plus watchlist and briefing scripts with optional Slack/webhook delivery.

## Tech stack

- Python, `requires-python >=3.12`; the core engine declares no runtime dependencies (`dependencies = []`).
- Dev tooling: `pytest>=9.1.0,<10` and `pytest-cov>=7,<8`, with a branch-coverage gate pinned at 84%.
- A separate Go MCP server (`mcp/`, `go.mod`) packaged as `.mcpb` bundles for Claude Desktop; requires local Python 3.12+.
- Integrates external CLIs (yt-dlp, arxiv-pp-cli, techmeme-pp-cli, digg-pp-cli) and third-party APIs (ScrapeCreators, Brave Search, Perplexity/OpenRouter).
- Distributed as an Agent Skill plus native plugin manifests for Claude Code, Codex, and Grok.

## When to reach for it

- You need recency-focused research on a person, company, product, or topic within a rolling 30-day window.
- You want community and market signal (Reddit threads, X posts, YouTube transcripts, Polymarket odds) rather than editorial web coverage alone.
- You are preparing for a meeting or sales call and want someone's recent public activity across platforms.
- You want to compare tools or discover topics before they peak, with cross-source numbers cited.

## When *not* to reach for it

- You need deep historical or archival research beyond the recent window (an `--as-of` lookback exists, but the tool is built around recency).
- You want a fully offline, zero-setup tool; many high-value sources require API keys, paid quotas, or logged-in browser sessions.
- You need deterministic, reproducible output; results are scored against live, changing engagement data.
- You are on Windows and want the Claude Desktop `.mcpb` bundle, which the README notes is deferred.

## Maturity signal

The skill is at version 3.17.0, MIT-licensed, and very actively maintained (last push mid-2026 against an early-2026 creation date), with the README documenting 175 merged pull requests across 15 releases — 122 of them from 52 community contributors. That points to an actively developed, community-driven project rather than a solo experiment, and the security section describes hardening work (stored-XSS fixes, supply-chain-hardened CI, scanners, an 84% coverage floor) done largely by outside contributors. Treat it as actively maintained with a broad contributor base.

## Alternatives

- Perplexity — use it instead when you want a hosted grounded-search product and would rather not wire up your own keys and browser sessions.
- GPT Researcher — use it instead when you want an open-source deep-research agent focused on web and document sources.
- Direct platform APIs or a general web-search MCP — use them instead when editorial coverage is enough and you do not need cross-platform engagement scoring.

## Notes

- The core Python engine intentionally ships with an empty runtime dependency list, keeping install friction low across many hosts.
- The project maintains both a Python engine and a separate Go MCP server, packaged as per-platform `.mcpb` bundles for Claude Desktop.
- Reddit access was rebuilt around keyless RSS and shreddit scraping after the public JSON API was removed, so real upvote scores and top comments still work without an API key.

## Tags

python, ai-agent, agent-skill, research, deep-research, web-search, social-media, llm, cli, mcp
