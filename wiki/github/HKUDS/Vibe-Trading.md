# HKUDS/Vibe-Trading

A natural-language finance research agent that gives an AI agent trading-oriented capabilities — multi-market backtesting, factor research, and gated broker connectivity — from a single install.

## What it is

Vibe-Trading is a Python agent, installable from PyPI as `vibe-trading-ai`, that turns natural-language requests into finance research, backtests, and (behind explicit safeguards) live trading actions. It targets developers and quantitatively minded users who want an LLM-driven agent wired to real market data, backtest engines, and broker APIs. The project ships a FastAPI backend, a React web UI, a CLI, and an MCP server, along with a large library of data loaders, backtest engines, and factor definitions.

## Key features

- Backtesting across multiple markets through dedicated engines (China A-share, China futures, crypto, forex, global equity/futures, India equity, and options portfolios).
- A large factor/alpha library (the "Alpha Zoo") plus tooling to register academic factors and monitor their IC/Sharpe decay over a lifecycle.
- Many market-data loaders with fallback chains (Yahoo, Binance, OKX, CCXT, Tushare, akshare, Longbridge, MetaTrader 5, and others) and 12 broker connectors.
- A multi-agent "swarm" mode and scheduled research, with results deliverable to 16 instant-messaging channel adapters (Telegram, Slack, Discord, Feishu, and more).
- An MCP server (supporting Streamable HTTP) exposing tools such as factor analysis and correlation.
- Live-trading safety controls described in the changelog: a consent-first mandate gate, kill switch, exposure caps, atomic daily order limits, and an AST-hardened backtest sandbox blocking network/subprocess/eval.

## Tech stack

- Python `>=3.11`; project version 0.1.11, MIT-licensed, development status Beta.
- Agent layer on LangChain `>=1.3.9,<2` and LangGraph `>=1.2.5,<1.3` with LangGraph checkpointing.
- Backend on FastAPI `>=0.104`, uvicorn, `sse-starlette`, pydantic v2, and `fastmcp>=2.14`; React 19 frontend (per README badges).
- Data/quant stack: pandas `>=2.0,<3.0`, numpy, scipy, scikit-learn, bottleneck, duckdb, `ccxt>=4.0`, `yfinance`, `akshare`, `tushare`, `smartmoneyconcepts`.
- Reporting/parsing: matplotlib, weasyprint, openpyxl, python-docx, python-pptx, pypdfium2, Pillow.
- Optional extras for individual brokers (ibkr, longbridge, mt5), LLM providers (deepseek, anthropic), data sources (ashare/baostock), and each IM channel.

## When to reach for it

- You want to run backtests and factor research by describing them in natural language instead of hand-writing pipelines.
- You need coverage across several markets (A-share, crypto, forex, options, India, global) from one framework.
- You want an agent connected to real data loaders and broker read APIs, with research pushed to chat channels.
- You want to expose finance tools to other agents over MCP.

## When *not* to reach for it

- You want hands-off automated real-money trading; live order paths are deliberately gated behind mandate confirmation and kill switches, and several broker connectors are read-only.
- You need a stable, production-grade trading system; the project is pre-1.0 Beta and iterating rapidly.
- You want a lightweight single-market backtester; the dependency surface and breadth are substantial.
- You are looking for investment advice or guaranteed returns — this is research/engineering tooling, not financial advice.

## Maturity signal

The repository has attracted roughly 26k stars since its April 2026 creation and pushes changes almost daily, with a detailed running news log and the last push landing the day before this snapshot — clearly active, heavily maintained development. It is MIT-licensed and not archived, at version 0.1.11 with a Beta classifier and only around 17 open issues, suggesting a small maintainer team keeping pace with a large contributor stream. Read it as young and fast-moving rather than settled; APIs and behavior are still changing.

## Alternatives

- Use **Microsoft Qlib** instead when you want an established quant research and factor platform without the LLM-agent layer.
- Use **Backtrader** or **Zipline** instead when you want conventional, well-documented Python backtesting and don't need the agent, data loaders, or broker breadth.
- Use **freqtrade** instead when your focus is specifically crypto algorithmic trading with a mature bot ecosystem.
- Use **FinGPT / FinRobot** instead when you want LLM-centered financial analysis with a different architecture and scope.

## Notes

- The README opens with a security warning that a named X account, Virtuals project, and token contract impersonate the project; the maintainers state they have never launched or endorsed any token or memecoin.
- Much of the recent changelog is correctness and safety hardening — look-ahead-bias fixes, causal rebalancing, fail-closed live state, and an external security audit whose ten findings were closed — indicating the maintainers treat backtest realism and live-trading safety as first-order concerns.

## Tags

python, trading, algorithmic-trading, backtesting, quantitative-finance, fintech, llm, ai-agent, multi-agent, mcp, fastapi
