# open-webui/open-webui

A user-friendly, self-hosted AI interface — the most-installed local LLM chat UI, drop-in for Ollama and any OpenAI-API-compatible backend.

## What it is

A web-based chat UI that fronts local LLM runtimes (Ollama is the canonical pairing) and any OpenAI-API-compatible endpoint. Provides multi-user accounts, chat history, model switching, RAG via document uploads, MCP integration, plugin / "function" extensions, and image generation hooks. Self-hosted by default; positioned as the open alternative to ChatGPT for users who want their data on their own hardware.

## Key features

- Drop-in Ollama integration — auto-discovers local Ollama models.
- OpenAI-API compatibility — works with any OpenAI-API-shaped backend (vLLM, TGI, OpenAI itself, hosted gateways).
- Multi-user with auth — proper user accounts, not single-tenant.
- RAG via document upload — drop PDFs/docs, get retrieval-augmented chat over them.
- MCP (Model Context Protocol) client support.
- Plugin / Function system for extending capabilities.
- Image generation integration via SD-WebUI or ComfyUI APIs.
- Mobile-friendly responsive UI.
- Self-hosted via Docker.

## Tech stack

- Python primary on the backend.
- SvelteKit on the frontend.
- SQLite default storage; Postgres optional.
- Docker-first deployment.

## When to reach for it

- You're running Ollama locally and want a polished chat UI in front of it.
- You want a multi-user ChatGPT-equivalent for a small team without sending data to a cloud LLM.
- You're running models in a regulated / air-gapped environment.

## When *not* to reach for it

- You're allergic to non-OSI licenses — SPDX is `NOASSERTION`; the project ships under a non-OSI custom license with specific commercial-use restrictions. Verify before SaaS-hosting.
- You want a developer-API-only surface — Open WebUI is a UI; for app integration go to the LLM runtime directly.
- You need a single-user-only minimal UI — LM Studio, Jan, or ChatALL fit lighter use cases.

## Maturity signal

139k stars, 20k forks, last push the morning this page was generated. 2.5-year-old project that became the canonical local-LLM chat UI essentially as it shipped. Open-issues count of 257 is low for the surface area — community triage is tight. The non-OSI license is the most-discussed limitation; the project's stewardship has tightened licensing as adoption scaled.

## Alternatives

- LibreChat — use when you want a strict-OSI-licensed alternative with similar feature set.
- LM Studio, Jan, ChatALL — use when you want a desktop chat app instead of a self-hosted web app.
- AnythingLLM — use when you want RAG-first chat with simpler config.

## Notes

The `NOASSERTION` license is the recurring concern — Open WebUI ships under a custom non-OSI license that restricts certain commercial-hosting cases. Most personal and internal use is fine; verify before reselling the hosted service. The Ollama-pairing is the most-installed combo and the easiest path to a working local LLM chat in <5 minutes.

## Tags

artificial-intelligence, large-language-model, chatbot, self-hosted, ollama, openai, python, sveltekit, retrieval-augmented-generation, model-context-protocol, user-interface
