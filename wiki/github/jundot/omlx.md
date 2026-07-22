# jundot/omlx

A local LLM inference server for Apple Silicon with continuous batching and tiered (RAM + SSD) KV caching, managed from the macOS menu bar.

## What it is

oMLX is an LLM inference server optimized for Apple Silicon Macs and built on Apple's MLX stack. It serves text LLMs, vision-language models, OCR models, embeddings, and rerankers behind an OpenAI- and Anthropic-compatible API, managed through a native menu-bar app or a CLI. Its distinguishing goal is persisting KV cache across a hot in-memory tier and a cold SSD tier, so long local coding sessions (for example with Claude Code) stay fast across requests and even across server restarts. It is for Mac developers who want to run local models with real caching and multi-model management.

## Key features

- **Tiered KV cache**: block-based management inspired by vLLM, with prefix sharing and copy-on-write; a hot RAM tier plus a cold SSD tier stored in safetensors that is restored on a matching prefix, even after a restart.
- **Continuous batching** through mlx-lm's BatchGenerator, with configurable max concurrent requests (default 8).
- **Multi-model serving** of LLMs, VLMs, OCR, embedding, and reranker models in one server, with LRU eviction, model pinning, per-model TTL, and a process memory limit to prevent system OOM.
- **API compatibility** with OpenAI and Anthropic (`/v1/chat/completions`, `/v1/messages`, `/v1/embeddings`, `/v1/rerank`, `/v1/models`), streaming usage stats, tool calling with per-family parsers, and JSON-schema structured output.
- **Admin dashboard** for monitoring, HuggingFace model download, one-click benchmarking, and per-model settings and profiles; all CDN assets are vendored for offline use.
- **Native Swift/SwiftUI menu-bar app** (not Electron) with auto-restart on crash, persistent stats, and Sparkle auto-update, plus one-click integrations for OpenClaw, OpenCode, Codex, Hermes Agent, Copilot, and Pi.

## Tech stack

- Python 3.11–3.13 (`requires-python = ">=3.11,<3.14"`); requires macOS 15.0+ and Apple Silicon (M1–M4).
- Built on MLX 0.32.0 (pinned), mlx-lm (git-pinned), mlx-vlm, and mlx-embeddings; optional native Metal custom kernels require full Xcode.
- Server layer: FastAPI >=0.108, uvicorn, httpx <1; huggingface-hub >=1.19; numpy <2.4.
- Optional extras: `mcp` (>=1.5) for Model Context Protocol, `grammar` (xgrammar for constrained decoding), `modelscope`, `audio` (mlx-audio), and `paroquant`.
- macOS app is native SwiftUI (Xcode 26.5+), packaged with venvstacks; a Homebrew formula is provided.
- Apache-2.0 license; pyproject marks it `Development Status :: 3 - Alpha`.

## When to reach for it

- You run local models on an Apple Silicon Mac and want an OpenAI- or Anthropic-compatible endpoint.
- You do long, tool-heavy local coding sessions and want KV cache reuse across requests and restarts.
- You want to serve mixed model types (LLM, VLM, OCR, embeddings, rerankers) from one server with memory-aware eviction.
- You prefer a menu-bar or web-managed server over editing config by hand.

## When *not* to reach for it

- You are not on Apple Silicon macOS — it is Mac-only and MLX-based, with no CUDA or Linux path.
- You need a stable production surface — pyproject marks it Alpha and it carries many pinned and git-pinned dependencies.
- You serve GLM-5.2, MiniMax M3, or Qwen3.5 families but cannot build the native kernels; the generic fallback is much slower and uses more memory (the README cites roughly 30x on GLM-5.2 prefill).
- You need a multi-GPU datacenter server — that is a different class of tool.

## Maturity signal

Created in February 2026, oMLX has roughly 18,000 stars and very recent activity (last push July 2026), so it is heavily used and rapidly developed. It is not archived and is Apache-2.0 licensed. However, pyproject marks it Alpha, it reports 711 open issues, and its dependencies are tightly pinned to an exact MLX ABI — so expect churn and breaking changes despite the momentum.

## Alternatives

- **mlx-lm (Apple)** — use instead when you only need the underlying MLX inference and CLI without the multi-model server, cache tiers, or GUI.
- **vllm-mlx** — oMLX started from vllm-mlx v0.1.0; use it if you want the smaller original MLX port that oMLX forked and extended.

## Notes

- Native custom kernels are not built by a plain `pip install -e .` and silently fall back to slower paths; building them needs the Metal toolchain from full Xcode, or you can use the official DMG, which ships them precompiled.
- The SSD cold tier means KV cache survives a server restart, not just a single session.
- The macOS app is native SwiftUI, not Electron, and installs a lightweight CLI shim so terminal commands and Apple Shortcuts can control the app-managed server.
- The custom-kernel binaries are ABI-coupled to the exact mlx 0.32.0 pin, so bumping MLX requires rebuilding them.

## Tags

python, swift, mlx, apple-silicon, macos, llm, inference-server, openai-api, machine-learning
