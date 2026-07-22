# esengine/DeepSeek-Reasonix

A DeepSeek-native AI coding agent for the terminal, distributed as a single static Go binary and tuned around DeepSeek's prefix cache to keep token costs low over long sessions.

## What it is

Reasonix is a command-line and TUI coding agent that drives a language model to edit code, run tools, and work through tasks in a project. It is built for developers who want a terminal-first agent whose behavior — providers, the active model, enabled tools, and plugins — is declared in a `reasonix.toml` config file rather than hardcoded. DeepSeek ships as a preset, but any OpenAI-compatible endpoint can be added as a configuration entry. The same local engine backs the CLI/TUI, an optional desktop app, and a VS Code extension.

## Key features

- Config-driven setup: providers, the agent, enabled tools, and plugins are all declared in `reasonix.toml`, with no hardcoded models.
- Multi-model and composable: any OpenAI-compatible endpoint is a config entry, and two models can optionally run together (executor plus planner) in separate cache-stable sessions.
- Plugin system where external tools run as subprocesses over stdio JSON-RPC (MCP-compatible), while built-in tools self-register at compile time.
- Cache-aware context maintenance: a small stable environment summary is injected at startup, and stale tool output is snipped or pruned before summary compaction.
- Single-binary distribution: a `CGO_ENABLED=0` static binary that cross-compiles to six targets (darwin, linux, windows × amd64, arm64) with one command, its only listed build dependency being a TOML parser.
- Editor integration via an ACP backend (`reasonix acp`), used by the VS Code / Open VSX extension for chat, editor context, and tool-call approvals.

## Tech stack

- Go (module `reasonix`), `go 1.25.0` with toolchain `go1.26.5`.
- TUI built on the Charm stack: `bubbletea/v2` v2.0.7, `bubbles/v2` v2.1.0, `lipgloss/v2` v2.0.4.
- Configuration via `BurntSushi/toml` v1.6.0 (the stated sole external dependency for distribution).
- Code intelligence via `tree-sitter/go-tree-sitter` v0.25.0 with grammars for JavaScript, Python, Rust, and TypeScript; syntax highlighting via `alecthomas/chroma/v2`; Markdown via `yuin/goldmark`.
- Shell handling via `mvdan.cc/sh/v3`, credential storage via `zalando/go-keyring`, plus SFTP, clipboard, and gitignore helpers.
- Distributed as a prebuilt native binary through npm (`reasonix`) and Homebrew; desktop builds and a VS Code extension reuse the same engine.

## When to reach for it

- You work primarily in a terminal and want a coding agent there rather than in a hosted web UI.
- You use DeepSeek, or any OpenAI-compatible endpoint, and want long sessions where prefix-cache stability keeps costs down.
- You want agent behavior, tools, and providers expressed declaratively in a config file that can be checked in and shared.
- You want to extend the agent with external tools over an MCP-compatible stdio plugin interface.
- You want one static binary you can drop onto multiple platforms without a runtime.

## When *not* to reach for it

- You do not use DeepSeek or an OpenAI-compatible API; other model backends are not described as supported.
- You want a fully hosted or browser-only agent with no local engine to install or configure.
- You prefer a zero-config tool and do not want to maintain a TOML configuration file to get provider, model, and tool wiring right.
- You need an IDE-native experience without also installing the CLI, since the VS Code extension starts and depends on a local `reasonix acp` backend.

## Maturity signal

The repository has drawn a very large audience in a short time — roughly 27.5k stars for a project created in April 2026 — and the last push is same-day as this snapshot, so development is active and fast-moving. It is MIT-licensed and not archived, with an unusually high open-issue count (over 1,200) and a `main-v2` default branch, both consistent with a young project under heavy, rapid iteration rather than a settled codebase. Treat it as actively maintained but still stabilizing.

## Alternatives

- Use **Aider** instead when you want an established, model-agnostic terminal coding agent with a large existing user base and pair-programming workflow.
- Use **OpenAI Codex CLI** instead when your work is centered on OpenAI models and you want that vendor's first-party terminal agent.
- Use **Gemini CLI** instead when you are on Google's models and want their native command-line agent.
- Use **Claude Code** instead when you want a terminal agent tuned around Anthropic's models and their tool-use conventions.

## Notes

- Distribution is deliberately minimal: the project markets itself as a single static Go binary whose only external dependency is a TOML parser, and it cross-compiles to six OS/arch targets from one command.
- The desktop app and VS Code extension are thin front-ends over the same local engine rather than separate implementations; the extension explicitly does not bundle the CLI and requires it to be installed first.

## Tags

go, cli, tui, ai-agent, coding-agent, llm, developer-tools, tool-use, prompt-caching, deepseek, mcp
