# AstrBotDevs/AstrBot

A self-hostable, plugin-extensible platform for running LLM-backed agent chatbots across many instant-messaging platforms.

## What it is

AstrBot is an open-source Python framework for building and hosting conversational AI agents that live inside chat platforms such as QQ, WeChat Work, Telegram, Slack, Discord, Feishu, and DingTalk. It connects those messaging platforms to LLM providers and agent backends, then adds the surrounding infrastructure a chatbot needs: conversation and context management, personas, a knowledge base, tool calling via MCP, and a plugin system. It targets individuals, developers, and teams who want to deploy a chatbot, customer-service assistant, automation helper, or knowledge base into their existing IM workflows rather than building that plumbing themselves.

## Key features

- Multi-platform messaging support with official adapters for QQ, OneBot v11, Telegram, WeChat Work, WeChat Official Accounts, Feishu, DingTalk, Slack, Discord, LINE, Satori, KOOK, Misskey, and Mattermost, plus community adapters for Matrix, Rocket.Chat, and VoceChat.
- Broad model-service coverage: direct LLM providers (OpenAI-compatible, Anthropic, Google Gemini, Moonshot, Zhipu, DeepSeek), self-hosted options (Ollama, LM Studio), and LLMOps platforms (Dify, Alibaba Cloud Bailian, Coze), alongside speech-to-text and text-to-speech services.
- Agent features including MCP tool use, skills, persona settings, a knowledge base, and automatic context compression.
- An agent sandbox for isolated execution of code and shell calls with session-level resource reuse.
- A plugin system with a marketplace of community plugins available for one-click installation.
- A web UI for administration and a web ChatUI with built-in agent sandbox and web search, plus internationalization support.

## Tech stack

- Python, requiring `>=3.12`.
- Web/async layer: `fastapi>=0.124.0`, `quart>=0.20.0`, `aiohttp>=3.11.18`, `websockets>=15.0.1`, `httpx[socks]`.
- Data and persistence: `sqlalchemy[asyncio]>=2.0.41`, `sqlmodel>=0.0.24`, `aiosqlite>=0.21.0`; vector search via `faiss-cpu`; retrieval via `rank-bm25` and `jieba`.
- LLM and agent SDKs: `openai>=1.78.0`, `anthropic>=0.51.0`, `google-genai>=1.56.0`, `dashscope`, and `mcp>=1.8.0,<2` for Model Context Protocol.
- Messaging-platform SDKs: `python-telegram-bot>=22.6`, `slack-sdk`, `py-cord` (Discord), `lark-oapi` (Feishu), `dingtalk-stream`, `wechatpy`, `qq-botpy==1.2.1`, and `aiocqhttp` (OneBot).
- Supporting libraries: `pydantic>=2.12.5`, `apscheduler` (scheduling), `aiodocker` (container control for the sandbox), `pypdf` and `markitdown-no-magika` (document ingestion), `cryptography`, `pyjwt`, and `pyotp`.
- Packaged with a `uv`-based install and an `astrbot` CLI entry point; built with `hatchling`, with a custom build hook that bundles a Vue dashboard; linted and formatted with `ruff`.
- Distributed under AGPL-3.0.

## When to reach for it

- You want a chatbot inside one or more IM platforms (QQ, Telegram, Slack, Discord, Feishu, DingTalk, and others) without writing the platform adapters yourself.
- You need a single deployment that can route between multiple LLM providers or LLMOps backends (OpenAI-compatible, Anthropic, Gemini, Dify, Coze, and others).
- You want agent behavior such as MCP tool calling, a knowledge base, personas, and sandboxed code/shell execution out of the box.
- You prefer self-hosting via `uv`, Docker, or a desktop app, and want a web UI to manage configuration.
- You expect to extend behavior through plugins rather than forking the core.

## When *not* to reach for it

- You only integrate with a single platform and a single model; the breadth of adapters and providers is overhead you will not use.
- You need a hosted, no-ops SaaS chatbot; AstrBot is self-managed infrastructure, and the deployment paths assume you run and maintain it.
- Your project cannot accept AGPL-3.0 terms, for example when embedding into closed-source distributed software.
- You are locked to a Python runtime older than 3.12, which the project requires.
- You want only a visual agent-workflow builder; AstrBot integrates platforms like Dify and Coze rather than replacing that class of tool.

## Maturity signal

AstrBot is actively maintained. The project dates to late 2022, carries a released version of 4.26.7 in its manifest, and shows a very recent last push, indicating ongoing development rather than maintenance-only status. Adoption signals are strong: roughly 37k stars and 2.6k forks point to a large user and contributor base, and the plugin marketplace and multi-language READMEs reinforce that. The open-issue count (over 1,300) is consistent with a busy, widely used project that fields substantial community traffic rather than a sign of neglect.

## Alternatives

- Dify — use instead when you want a standalone visual LLM app and workflow builder and do not need to host a bot inside IM platforms; AstrBot can also consume Dify as a backend.
- Coze — use instead when you prefer a managed agent-building platform over self-hosted infrastructure; AstrBot integrates it as one of several agent backends.
- NapCatQQ — use instead when you need only a QQ protocol implementation at the transport layer, not a full multi-platform agent framework.
- MaiBot — use instead when your focus is a QQ-centered conversational companion bot rather than a general multi-platform agent host.

## Notes

- Despite the name and IM focus, the codebase includes a "computer"/sandbox layer with browser, filesystem, GUI, shell, and Python "olayer" modules and multiple sandbox booters (including local, boxlite, and shipyard variants), suggesting agent execution is a substantial part of the system beyond simple chat.
- Plugins are referred to internally as "stars"; the repository ships built-in stars and builtin command plugins under `astrbot/builtin_stars`.
- The package bundles a Vue dashboard that is not tracked in version control but is built and included at package time via a custom hatch build hook.
- The description advertises the project as an alternative to a general AI assistant, and the README frames the goal as a bot that both provides companionship and reliably completes tasks.

## Tags

python, llm, chatbot, agent, framework, self-hosted, mcp, rag, knowledge-base, sandboxing, plugin-system, instant-messaging
