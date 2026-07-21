# Panniantong/Agent-Reach

A capability layer that gives an AI agent read and search access to web platforms through one CLI, without paying for platform APIs.

## What it is

Agent Reach is a Python CLI that lets an AI coding agent read and search sources like the web, YouTube, RSS, GitHub, Twitter/X, Reddit, Bilibili, Xiaohongshu, and more. Rather than reimplementing readers, it selects, installs, health-checks, and routes to upstream tools, so the agent calls those tools directly. It targets developers who repeatedly set up internet access for agents and want the currently most reliable path chosen and configured for them.

## Key features

- Per-platform backend routing: each channel is an ordered list of a preferred backend plus fallbacks, with real probing to pick the first fully working one.
- `agent-reach doctor` reports each channel's status and which backend is currently active.
- Zero-configuration channels for web (Jina Reader), YouTube (yt-dlp), RSS (feedparser), GitHub (gh CLI), and full-web search (Exa via mcporter, no API key).
- Login-state channels (Twitter, Reddit, Xiaohongshu, Facebook, Instagram) with credentials stored locally only.
- Installs a SKILL.md into the agent's skills directory so the agent knows which upstream tool to call for a given need.
- Install and uninstall support `--safe` and `--dry-run` modes to avoid or preview system changes.
- MCP server integration for exposing the toolset.

## Tech stack

- Python `>=3.10` (classifiers list 3.10 through 3.12), packaged with hatchling; exposes the `agent-reach` console script.
- Core dependencies: `requests>=2.28`, `feedparser>=6.0`, `python-dotenv>=1.0`, `loguru>=0.7`, `pyyaml>=6.0`, `rich>=13.0`, `yt-dlp>=2024.0`.
- Optional extras: `playwright>=1.40` (browser), `browser-cookie3>=0.19` (cookies), `mcp[cli]>=1.0` (all).
- Upstream tools it routes to include Jina Reader, yt-dlp, gh CLI, bili-cli, OpenCLI, rdt-cli, xiaohongshu-mcp, linkedin-mcp, and Exa via mcporter.

## When to reach for it

- Giving an AI coding agent (Claude Code, OpenClaw, Cursor, Windsurf) read and search access across many platforms in one setup.
- Bootstrapping a fresh agent environment quickly through a single install instruction.
- Situations where a specific platform backend keeps breaking and you want managed fallback routing instead of re-integrating by hand.
- Keeping platform credentials on your own machine while still letting the agent read login-gated content.

## When *not* to reach for it

- You need to operate web pages (form submission, login flows, multi-account sessions) rather than read them; the README points to BrowserAct for that.
- Your agent cannot execute shell commands (for example an OpenClaw setup without the exec/coding profile).
- You require guaranteed stability; the free upstream sources are subject to rate limits, blocking, and interface changes.
- You are on a server that needs a proxy — the README notes that only server deployments may need one (about $1/month).

## Maturity signal

At roughly 59k stars, created in early 2026 and pushed within the last week of these facts, MIT-licensed and marked "4 - Beta" at version 1.5.0, this is a young but very actively maintained and widely adopted project. The design around swappable backends and a real 2026-06 backend migration (yt-dlp to bili-cli for Bilibili) suggests ongoing, hands-on upkeep rather than a static release.

## Alternatives

- **yt-dlp / gh CLI (the upstream tools directly)** — use them when you only need one platform and do not want the routing and health-check layer on top.
- **BrowserAct** — use it when you need interactive browser automation (logins, forms, workflows) instead of reading content.
- **A hosted scraping or search API** — reach for one when you want managed reliability and are willing to pay, rather than routing free upstream tools yourself.

## Notes

Agent Reach describes itself as a capability layer, not a wrapper: the actual reading and searching are done by the agent calling upstream tools directly, with no intermediate wrapper. Channels store cookies and tokens locally in `~/.agent-reach/config.yaml` with 600 permissions, and the README recommends using a dedicated secondary account for cookie-based platforms because scripted access carries a ban risk. Installation is triggered by pasting a URL to an install document into the agent, which then self-installs.

## Tags

python, cli, ai-agent, web-scraping, mcp, llm-tools, automation, social-media, rss, search
