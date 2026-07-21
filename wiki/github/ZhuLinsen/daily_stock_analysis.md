# ZhuLinsen/daily_stock_analysis

An LLM-driven multi-market stock analysis system that aggregates market data and news, generates decision reports, and pushes them to chat channels on a schedule.

## What it is

This project analyzes a personal watchlist across several equity markets and produces AI-generated decision reports, then pushes a "decision dashboard" to your chat channels on a schedule. It pulls from multiple market-data and news sources, and can run at zero cost through a forked GitHub Actions workflow. It targets individual investors and tinkerers who want automated daily analysis and notifications rather than a paid, hosted service, and it is explicitly for study and research, not investment advice.

## Key features

- LLM decision reports per stock: core conclusion, score, trend, buy/sell levels, risk alerts, catalysts, and an action checklist.
- Multi-market coverage of A-share, Hong Kong, US, Japan, Korea, and Taiwan equities plus ETFs, with prioritized data-source fallback.
- Multi-source news and sentiment search across Anspire, SerpAPI, Tavily, Bocha, Brave, MiniMax, and self-hosted SearXNG.
- Push to WeCom, Feishu, Telegram, Discord, Slack, and email; scheduled runs via GitHub Actions, Docker, or local tasks.
- A web and desktop workspace for manual analysis, task progress, history, backtesting, holdings, and configuration, with light and dark themes.
- Agent strategy Q&A with 15 built-in strategies (moving-average crossovers, Chan theory, Elliott wave, event-driven, growth, and others) and multi-turn follow-ups.
- Smart import from images, CSV/Excel, and clipboard, with code, name, and pinyin autocomplete.

## Tech stack

- Python 3.10+ (black targets 3.10 through 3.12); a FastAPI service with uvicorn, SQLAlchemy, `schedule`, and `exchange-calendars`.
- Unified LLM access via `litellm` (Gemini, Anthropic, OpenAI-compatible, DeepSeek, Qwen, Ollama, and hosted routers Anspire and AIHubMix).
- Market data from `efinance`, `akshare`, `tushare`, `pytdx`, `baostock`, `yfinance`, `longbridge`, `tickflow`, and `futu-api`, ordered by fallback priority.
- News search via `tavily-python` and `google-search-results` (SerpAPI); notifications via `lark-oapi`, `dingtalk-stream`, and `discord.py` with `PyNaCl`.
- Reporting with `jinja2`, `markdown2`, and `imgkit` (requires wkhtmltopdf); data handling with `pandas` and `numpy`.
- A React/Vite web app (`apps/dsa-web`) and an Electron desktop app (`apps/dsa-desktop`); a bundled AlphaSift screening engine pulled as a git dependency.

## When to reach for it

- Automated daily analysis and push of a personal multi-market watchlist.
- Zero-cost scheduled runs by forking the repository and enabling GitHub Actions.
- Conversational, strategy-based stock querying backed by LLMs.
- Self-hosting a dashboard plus notifications instead of subscribing to a service.

## When *not* to reach for it

- You need order execution or actual trading; this is analysis and notification only, and the README disclaims any investment advice.
- You need sub-second or real-time quant trading; the system is daily-oriented and free sources are rate-limited.
- Your primary need is screening or backtesting; the author points to the companion AlphaSift and AlphaEvo projects instead.
- You require guaranteed data reliability without configuring token-based data sources.

## Maturity signal

At roughly 58k stars, about 50k forks, CI enabled, an MIT license, and pushes within the last day of these facts, this is a young but very actively maintained project. The fork count nearly matching the star count reflects the fork-and-run GitHub Actions deployment model rather than heavy code contribution.

## Alternatives

- **ZhuLinsen/alphasift** — use it when you need multi-factor screening and whole-market scanning to pick candidates from a pool.
- **ZhuLinsen/alphaevo** — use it when you need strategy backtesting and iterative parameter or combination evolution.
- **AkShare / Tushare directly** — use them when you only need raw market data and will build your own analysis and delivery.

## Notes

The near-1:1 fork-to-star ratio is a deployment artifact: users fork and run the project via GitHub Actions. Free built-in data sources (AkShare, Baostock, YFinance) work with zero configuration but are not stability-guaranteed, so token sources such as TickFlow, Tushare, and Longbridge are recommended for long-term or batch use. Beyond the CLI and Actions flow, the repository ships a React web workspace and an Electron desktop app, and it belongs to a three-project family alongside AlphaSift (screening) and AlphaEvo (backtesting).

## Tags

python, llm, ai-agent, quantitative-finance, fastapi, docker, stock-analysis, litellm, notifications, react
