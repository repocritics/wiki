# RightNow-AI/openfang

An open-source "agent operating system" written in Rust that compiles to a single binary and runs autonomous agents on schedules.

## What it is

OpenFang is a self-hosted runtime for autonomous LLM agents, built in Rust and shipped as a single binary. Rather than only answering when prompted, it runs pre-packaged autonomous capabilities called "Hands" on schedules — for research, lead generation, target monitoring, social media management, and similar recurring work — and reports to a local dashboard. It targets developers who want to run long-running, self-directed agents locally instead of assembling a Python agent stack.

## Key features

- **Hands**: autonomous capability packages that run independently on schedules. Seven are bundled (Clip, Lead, Collector, Predictor, Researcher, Twitter, Browser), each defined by a `HAND.toml` manifest, a multi-phase system prompt, a `SKILL.md`, and approval guardrails.
- **40 channel adapters** (Telegram, Discord, Slack, WhatsApp, Signal, Matrix, email, and more) with per-channel model overrides, DM/group policies, rate limiting, and output formatting.
- **27 LLM providers** reached through 3 native drivers (Anthropic, Gemini, OpenAI-compatible), with routing by task-complexity scoring, automatic fallback, and cost tracking.
- **OpenAI-compatible API**: 140+ REST/WS/SSE endpoints and a built-in web dashboard at `http://localhost:4200`.
- **Security systems** (16 listed), including a WASM sandbox with fuel metering, a Merkle hash-chain audit trail, Ed25519-signed agent manifests, SSRF protection, and a GCRA rate limiter.
- **Memory and migration**: SQLite persistence with vector embeddings, plus a migration engine that imports agents, history, skills, and config from OpenClaw, LangChain, and AutoGPT.

## Tech stack

- Rust (edition 2021, `rust-version = 1.75`), organized as a Cargo workspace of 14 crates.
- Async runtime `tokio` 1; HTTP server `axum` 0.8 with `tower`/`tower-http`; HTTP client `reqwest` 0.12 over `rustls`.
- WASM sandbox via `wasmtime` 43.
- Storage via `rusqlite` 0.31 (bundled SQLite).
- Security stack: `ed25519-dalek` 2, `aes-gcm` 0.10, `argon2` 0.5, `hmac`, `sha2`, `zeroize`, and `governor` 0.10 for rate limiting.
- MCP via `rmcp` 1.2 (the official Rust SDK); CLI/TUI via `clap` 4 and `ratatui` 0.29.
- Email through `lettre` 0.11 and `imap`; MQTT through `rumqttc`.
- Desktop app built on Tauri 2.0; optional WhatsApp Web gateway needs Node.js >= 18.
- Workspace `Cargo.toml` declares license `Apache-2.0 OR MIT`.

## When to reach for it

- You want agents that run on a schedule and act on their own (monitoring, research, lead generation), not just interactive chat.
- You prefer a single self-hosted binary over managing a Python environment and its dependencies.
- You need many messaging-channel integrations available out of the box.
- You want a local OpenAI-compatible endpoint backed by agent routing and cost tracking.

## When *not* to reach for it

- You need a stable, pinnable API: the project is pre-1.0 and its own notice warns of breaking changes between minor versions, recommending you pin to a specific commit for production.
- Your team extends agents in Python and expects that ecosystem; the core is Rust.
- You only need a simple prompt/response chatbot — the operating-system model is heavier than that use case requires.
- You depend on features from the less-mature Hands; the README flags Browser and Researcher as the most tested, implying others are less so.

## Maturity signal

Created in February 2026, OpenFang has drawn roughly 18,000 stars in a few months and receives frequent pushes (last push July 2026), so it is actively developed with strong early traction. But it describes itself as feature-complete yet pre-1.0 (Cargo.toml version 0.6.9), and its own stability notice warns of breaking changes and advises pinning to a commit for production — so treat it as fast-moving rather than API-stable. Licensing is stated inconsistently across sources: repository metadata lists Apache-2.0, the workspace manifest declares `Apache-2.0 OR MIT`, and the README's license section says MIT.

## Alternatives

- **LangGraph** — use instead when you are already in the Python/LangChain ecosystem and want graph-structured orchestration.
- **CrewAI** — use instead when you want a Python multi-agent framework organized around role-based crews.
- **AutoGen** — use instead when you want a Python conversational multi-agent framework and Docker-based isolation.

## Notes

- The entire system, including the bundled Hands and skills, compiles into one binary the README cites at about 32 MB — no pip install or Docker pull to add capabilities.
- The comparison tables and benchmark charts in the README ("Measured, Not Marketed") are self-reported; the numbers warrant independent verification.
- Version numbers are inconsistent within the README itself (a v0.5.10 notice alongside a 0.6.9 badge and a 0.6.9 workspace version).

## Tags

rust, ai-agents, agent-framework, llm, mcp, cli, wasm, sandboxing, self-hosted, openai-api
