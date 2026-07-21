# aaif-goose/goose

An open-source, extensible AI agent — desktop app, CLI, and API — that runs on your machine, works with many LLM providers, and connects to tools through the Model Context Protocol.

## What it is

goose is a general-purpose AI agent that runs locally and goes beyond code suggestions to install, execute, edit, and test with any LLM. It ships as a native desktop app for macOS, Linux, and Windows, a full CLI for terminal workflows, and an API to embed it elsewhere, and is built in Rust. Beyond coding, it is positioned for research, writing, automation, and data analysis. goose is part of the Agentic AI Foundation (AAIF) at the Linux Foundation.

## Key features

- Three surfaces from one project: a desktop app, a CLI, and an embeddable API.
- Works with 15+ providers (Anthropic, OpenAI, Google, Ollama, OpenRouter, Azure, Bedrock, and more) via API keys or existing subscriptions through ACP.
- Connects to 70+ extensions through the Model Context Protocol open standard.
- Custom Distributions: build your own goose distro with preconfigured providers, extensions, and branding.
- Recipe, schedule, session, and review/orchestrator commands are present in the CLI source tree.
- Local model inference support alongside remote providers.

## Tech stack

- Rust, edition 2021, workspace version 1.44.0, `rust-version = 1.94.1`; a Cargo workspace over `crates/*`.
- Agent and protocol layers: `rmcp` 1.4, `agent-client-protocol` 1.x (ACP), `axum` 0.8, `tokio` 1.48, `reqwest`, `rustls`.
- Code understanding via `tree-sitter` parsers for Go, Java, JavaScript, Kotlin, Python, Ruby, Rust, Swift, and TypeScript.
- Local inference via pinned `llama-cpp-2` (=0.1.146) plus `candle-core`/`candle-nn`; observability via the OpenTelemetry crates.
- Desktop tooling pinned through hermit (Node, pnpm, cmake, protoc); Apache-2.0 licensed.

## When to reach for it

- You want a local, general-purpose agent available as a desktop app, a CLI, and an API from a single project.
- You want to stay provider-agnostic and switch among many LLMs, including via existing Claude, ChatGPT, or Gemini subscriptions.
- You rely heavily on MCP extensions and want broad extension support.
- You need to ship a branded, preconfigured agent distribution to a team.

## When *not* to reach for it

- You want a purely cloud-hosted or SaaS agent rather than something running on your machine.
- You want a minimal single-binary tool with no provider or extension setup.
- You only need inline IDE code completions; goose is scoped to run tasks end to end, which is heavier.
- You prefer a Python-native library you can embed directly; goose's core is Rust.

## Maturity signal

goose has been developing since August 2024, is at workspace version 1.44.0, is Apache-2.0 licensed, and is under Linux Foundation governance as part of the AAIF, with recent activity (last push mid-2026). The combination of formal governance, a maintainers file, a health-score badge, and a large, actively churning source tree points to a mature, well-governed project rather than an early experiment. Treat it as actively maintained and organizationally backed.

## Alternatives

- Aider — use it instead when you want a focused terminal-based pair programmer scoped to a git repository.
- Claude Code — use it instead when you want a vendor-integrated agent CLI tied to one provider's models.
- Cline — use it instead when you want an agent embedded directly in VS Code rather than a standalone desktop/CLI app.

## Notes

- Despite the coding-agent framing, the README explicitly positions goose as general-purpose, for research, writing, automation, and data analysis, not only code.
- The build is tuned for developer iteration: a Cargo profile override drops debug info for third-party dependencies (but not workspace crates) to shrink debug builds and speed linking.
- The workspace vendors V8 and pins several transitive crates (icu, temporal) to work around upstream version constraints.

## Tags

rust, ai-agent, cli, desktop-app, llm, mcp, acp, developer-tools, automation, local-first
