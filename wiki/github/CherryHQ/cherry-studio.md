# CherryHQ/cherry-studio

An AI productivity studio with chat, autonomous agents, and 300+ pre-built assistants — unified access to frontier LLMs in one desktop app.

## What it is

A TypeScript desktop app that combines multi-provider LLM chat (Claude, GPT, Gemini, DeepSeek, etc.), a 300+ catalog of pre-built "assistants" (system-prompted personas), and an autonomous-agent layer. Distributed at cherryai.com. AGPL-3.0 licensed — the network-service copyleft.

## Key features

- Multi-provider LLM chat: OpenAI, Claude, Gemini, DeepSeek, Groq, OpenRouter, Ollama, etc.
- 300+ pre-built assistants — domain-specific system prompts.
- Autonomous agent layer with skills (per the topic tags: claude-code, codex, hermes-agent ecosystem).
- Cross-platform desktop (Windows, macOS, Linux).
- AGPL-3.0 licensed.

## Tech stack

- TypeScript primary.
- Electron-style desktop packaging.

## When to reach for it

- You want one desktop chat client across many LLM providers + a curated assistant catalog.
- You want a unified UX for chat + agent workflows.

## When *not* to reach for it

- You're allergic to AGPL — the copyleft pulls SaaS-style redistribution into source-release.
- You want vendor-supported chat — the official providers' apps are simpler.
- You're risk-averse about installing third-party desktop apps that handle LLM API keys; audit the source before configuring credentials.

## Maturity signal

47k stars, 4k forks, AGPL-3.0, actively maintained. The topic-list overlap with multiple emerging agent ecosystems (claude-code, codex, hermes-agent, openclaw, skills) signals broad-platform aspiration. The 1,137 open-issues count is moderate.

## Alternatives

- `open-webui/open-webui` — self-hosted web UI alternative.
- `ChatGPTNextWeb/NextChat` — lighter cross-platform chat client.
- LibreChat — strict-OSI alternative.

## Tags

typescript, large-language-model, chatbot, agent, claude, openai, gemini, desktop, agpl, cross-platform
