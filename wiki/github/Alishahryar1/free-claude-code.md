# Alishahryar1/free-claude-code

A local proxy that lets coding agents such as Claude Code, Codex, and Pi run against your choice of cloud or local model providers.

## What it is

Free Claude Code (FCC) is a self-hosted proxy that connects coding agents to OpenAI-compatible AI providers. It runs on your machine, exposes a local Admin UI for picking and validating providers, and launches Claude Code, Codex, or Pi so they talk to the model you configured instead of a default backend. It is for developers who want to route their coding agents to free, paid, or local models and switch between them from one place.

## Key features

- Launchers `fcc-claude`, `fcc-codex`, and `fcc-pi` that start each coding agent against the current Admin UI settings, with normal CLI arguments still working.
- A local Admin UI to choose and validate among 25 listed cloud and local providers, with each model slug built as `<provider-id>/<exact-provider-model-id>`.
- Per-tier model routing: `MODEL` acts as the fallback, while `MODEL_FABLE`, `MODEL_OPUS`, `MODEL_SONNET`, and `MODEL_HAIKU` can override individual Claude Code tiers.
- Preserves streaming, tool use, reasoning, and image input across compatible models, with configurable reasoning-effort handling.
- Editor integrations for Claude Code and Codex in VS Code, and Claude Code through JetBrains ACP.
- Optional Discord and Telegram bots for running Claude Code sessions, including voice-note transcription, plus optional token authentication for the proxy.

## Tech stack

- Primary language: Python, requiring Python `>=3.14.0`.
- Web/proxy layer: FastAPI `>=0.139.2` and Uvicorn `>=0.51.0`, with httpx `>=0.28.1`, aiohttp, and the OpenAI client `>=2.46.0`.
- Config and validation: Pydantic `>=2.13.4`, pydantic-settings, python-dotenv, jsonschema; token counting via tiktoken.
- Messaging: python-telegram-bot `>=22.8` and discord.py `>=2.7.1`; logging via Loguru.
- Optional voice extras: NVIDIA Riva client for NIM transcription, or torch/transformers/accelerate/librosa for local Whisper.
- Tooling: hatchling build backend, uv (`>=0.11.16`), pytest, Ruff, and the Ty type checker; packaged at version 4.12.0. Desktop tray uses pystray and Pillow on Windows and macOS.

## When to reach for it

- Running Claude Code, Codex, or Pi against a specific provider or a local model server (LM Studio, llama.cpp, Ollama).
- Comparing or switching among many providers without hand-editing each agent's configuration.
- Assigning different Claude Code tiers to different models to balance cost and capability.
- Driving a coding agent from Discord or Telegram, optionally with voice notes.

## When *not* to reach for it

- You are happy using a coding agent with its default first-party backend and do not need provider switching.
- You cannot run Python 3.14, which is a hard requirement.
- You need a provider or model that is not OpenAI-compatible or not tool-capable, since coding agents rely on tool calls.
- You want a fully hosted service; FCC is a local proxy you run and keep running yourself.

## Maturity signal

The repository is only a few months old yet has accumulated a very large star count, indicating fast early traction. A recent push date and a moderate open-issue count suggest active development keeping pace with that interest. The MIT license is permissive, and the project is not archived. The strict Python 3.14 floor and frequently updated provider list signal a fast-moving codebase that tracks current model releases closely.

## Alternatives

- LiteLLM — reach for it when you want a provider-agnostic proxy/SDK as library or gateway infrastructure rather than a desktop-oriented launcher for coding agents.
- OpenRouter — reach for it when you prefer a hosted aggregation endpoint over running a local proxy of your own.
- Ollama — reach for it when your goal is simply serving local models and you do not need multi-provider routing or agent launchers.

## Notes

- Some providers have setup caveats the README calls out explicitly: Kimi Code subscription keys are for personal interactive coding-agent use, Amazon Bedrock uses a Mantle OpenAI-compatible endpoint, and Vertex AI uses Google Application Default Credentials instead of an API key.
- The installer is a single shell or PowerShell command that also serves as the update mechanism when re-run, and the uninstaller removes FCC while leaving uv, Python, and the coding agents intact.

## Tags

python, proxy, cli, coding-agents, llm, openai-compatible, fastapi, model-routing, self-hosted, developer-tools, local-models
