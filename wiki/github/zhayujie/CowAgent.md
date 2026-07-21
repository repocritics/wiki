# zhayujie/CowAgent

An open-source personal AI assistant and agent harness that plans tasks, runs tools and skills, and self-evolves with memory and knowledge across many chat channels.

## What it is

CowAgent (formerly chatgpt-on-wechat) is a self-hosted Python agent that proactively plans and executes multi-step tasks, controls a computer and external services through tools, and builds a personal knowledge base and long-term memory. It plugs into any major LLM provider and serves multiple messaging channels in parallel from a single instance. It is aimed at individuals and teams who want a 24/7 assistant they can run on a personal computer or server, managed from a unified web console.

## Key features

- Task planning that decomposes work and loops over tools until the goal is reached.
- Three-tier long-term memory (context, daily, `MEMORY.md`) with a nightly "Deep Dream" distillation pass and hybrid keyword-plus-vector retrieval.
- A personal knowledge base that auto-curates structured Markdown with a knowledge-graph view, plus Self-Evolution that reviews conversations to improve skills and consolidate memory.
- Built-in tools for file I/O, terminal (`bash`), browser automation, scheduling, web search, web fetch, vision, and memory, with native MCP integration via `mcp.json`.
- A Skills system with one-click install from Skill Hub, GitHub, or ClawHub, plus conversational skill authoring.
- Channels for Web console, Telegram, Slack, Discord, WeChat, Feishu, DingTalk, WeCom, QQ, and more, with multimodal text, image, voice, and file support.

## Tech stack

- Python 3.7+ per `pyproject.toml`; the `cow` CLI is built on Click 8.0+ and `requests`.
- Web console served on port 9899; one-line installer for Linux/macOS, Windows PowerShell, and Docker.
- Channel SDKs: `python-telegram-bot`, `slack_bolt`, `discord.py`, `wechatpy`, `lark-oapi` (>=1.5.5), `dingtalk_stream`, `websocket-client` (>=1.4.0).
- Model SDKs: `dashscope` (Qwen) and `zai-sdk` (GLM); OpenAI, Claude, Gemini, DeepSeek, and others configured from the console.
- Supporting libraries: `numpy` (>=1.21), `aiohttp` (>=3.8.6,<3.10), `Pillow`, `croniter`, `PyYAML`, `pycryptodome`, `json-repair`, `qrcode`.
- Python 3.13+ support via a GitHub build of `web.py` plus `legacy-cgi`.

## When to reach for it

- You want a self-hosted, always-on personal assistant reachable from Chinese and international messaging platforms alike.
- You need multi-step task planning combined with persistent memory and an auto-curated knowledge base.
- You want to swap LLM providers freely and route chat, vision, image generation, ASR/TTS, and embeddings to different vendors.
- You want to install or author skills and integrate MCP tools without heavy configuration.

## When *not* to reach for it

- You want a stateless chatbot; the agent's planning, memory, and evolution add substantial token cost, which the README flags explicitly.
- You cannot provide a trusted environment — the agent has access to your local operating system.
- You need only a single web chat interface and none of the channel or automation machinery.
- You require a non-Python runtime or a fully managed hosted service (offered separately via LinkAI, not this repo).

## Maturity signal

Created in August 2022 and pushed within a day of this writing, CowAgent is the most established project of its kind here, with a very large star and fork count and a continuous release cadence (currently the 2.1.x line). Its history as chatgpt-on-wechat, a detailed changelog, and MIT licensing point to a mature, actively maintained codebase that has evolved from a WeChat chatbot into a general agent harness. The large fork count signals heavy real-world reuse.

## Alternatives

- HKUDS/nanobot — use it when you want a smaller, more minimal self-hosted agent core with a similar channel and MCP surface.
- zhayujie/bot-on-anything — use it when you want a lightweight LLM-to-messaging bridge without planning, memory, or a knowledge base (from the same author).
- AgentMesh — use it when your problem needs a multi-agent team framework rather than a single personal assistant.

## Notes

The project was renamed from chatgpt-on-wechat; the old GitHub URL redirects, and existing users can optionally update their git remote. Despite the WeChat origin, `requires-python` in `pyproject.toml` is still 3.7+ while the requirements pin newer libraries and add Python 3.13 workarounds, so the effective supported range is narrower than the manifest alone suggests.

## Tags

python, ai-agent, llm, chatbot, mcp, self-hosted, multi-agent, memory, knowledge-base, skills, multimodal
