# HKUDS/nanobot

A small, self-hosted personal AI agent runtime that connects one agent core to chat apps, tools, memory, and deployment targets.

## What it is

nanobot is an open-source Python runtime for running your own long-lived AI agent. It keeps a small, readable agent loop at the center and adds the practical pieces needed for real work: a bundled browser WebUI, chat-app channels, tools, memory, MCP support, model routing, and automation. It is aimed at individuals and developers who want to own and self-host an agent rather than depend on a hosted platform.

## Key features

- Runs in a bundled browser WebUI (shipped inside the published wheel) or in the terminal.
- Connects to Telegram, Discord, Slack, WeChat, Email, Mattermost, DingTalk, and other chat apps.
- Built-in tools including files, shell, web search, web fetch, MCP, cron, image generation, and subagents.
- Session history plus long-term memory via "Dream", and support for long-horizon goals and scheduled automations.
- Exposes a Python SDK and an OpenAI-compatible API for integrations.
- Named model presets with `/model` switching and `fallbackModels`, across OpenAI-compatible and local providers.

## Tech stack

- Python 3.11+; packaged as `nanobot-ai` (v0.2.2) built with Hatchling.
- Typer for the CLI; Pydantic v2 and pydantic-settings for configuration.
- `anthropic`, `openai`, and `mcp` client libraries; `tiktoken` for tokenization.
- `websockets` / `websocket-client` for the gateway and WebUI channel; `httpx` for HTTP.
- `ddgs` for search, `readability-lxml` for content extraction, and bundled document readers (`pypdf`, `python-docx`, `openpyxl`, `python-pptx`).
- Optional extras for Azure, Bedrock, and Langfuse observability; the source-tree WebUI builds with `bun` or `npm`.

## When to reach for it

- You want a self-hosted personal agent you can inspect, customize, and run 24/7 on your own machine or server.
- You want to reach the agent from familiar chat apps rather than a single web interface.
- You need long-running goals, scheduled automations, and persistent memory.
- You want to integrate an agent into your own code through a Python SDK or an OpenAI-compatible API.

## When *not* to reach for it

- You need a fully managed, hosted agent with no infrastructure to run.
- You require production-grade stability guarantees; the package is marked Development Status: Alpha.
- Your use case is a one-off script or a stateless chatbot without memory, channels, or automation.
- You cannot run Python 3.11+ or provide model-provider credentials.

## Maturity signal

Created in February 2026 with commits pushed within a day of this writing and a high commit-activity signal, nanobot is actively and heavily developed. The large open-issue count and the explicit "3 - Alpha" classifier indicate a fast-moving project still stabilizing its interfaces, so features and configuration may shift between releases. The permissive MIT license and a "Durability Release" changelog suggest reliability is a current focus.

## Alternatives

- CowAgent — use it when you want a comparable multi-channel personal agent with a built-in knowledge base and self-evolution features.
- LibreChat — use it when you primarily need a self-hosted multi-provider chat UI rather than a tool-using agent runtime.
- LangChain or LangGraph — use these when you are building a bespoke agent from library components rather than running a batteries-included agent.

## Notes

The WebUI is bundled inside the published wheel, so a PyPI or `uv` install needs no separate build step, while a from-source install requires `bun` or `npm` to build it. A Render blueprint (`render.yaml`) deploys the gateway and WebUI as one service, but persistent memory requires a paid disk tier.

## Tags

python, ai-agent, self-hosted, chatbot, llm, mcp, cli, automation, memory, openai-compatible
