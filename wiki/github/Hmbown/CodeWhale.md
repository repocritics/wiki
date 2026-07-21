# Hmbown/CodeWhale

A terminal coding agent written in Rust that runs any model — open models first — through a single runtime and toolset.

## What it is

Codewhale is a command-line coding agent that reads your code, edits files, runs commands, checks its own work, and stops when the task is done or it needs input. It is model-agnostic: you give it a provider, a model, and a task, and can switch models mid-task with `/model`. It began as `deepseek-tui` and was generalized so that the model is a swappable component rather than the product, aimed at developers who want one runtime across hosted, gateway, and local model routes.

## Key features

- One runtime and toolset across DeepSeek, Claude, GPT, Kimi, GLM, and 30+ providers, plus local vLLM, SGLang, or Ollama with no key required; context budgets and prices come from the real route, and an unknown price is shown as unknown rather than $0.
- Safety by construction: Plan mode is read-only, approvals gate risky calls, and an OS sandbox uses Seatbelt, Landlock, seccomp, or bwrap; a repo `constitution.json` compiles into write holds that even Full Access cannot skip.
- Fleets that record every step in an append-only ledger, with `fleet resume` to pick up where you stopped and a per-turn receipt you can inspect.
- Multiple entry points: an interactive TUI, `codewhale exec` for scripts and CI, and `codewhale web` for a loopback-only browser client on 127.0.0.1.
- In-TUI controls: `/model` to switch provider and model together, `/fleet` to run a team of workers, `/restore` to undo a turn, `Tab` to cycle Plan/Act/Operate, `Shift+Tab` to cycle Ask/Auto-Review/Full Access approval postures, and `!` to run a shell command through the normal approval path.

## Tech stack

- Rust, edition 2024, `rust-version = "1.88"` (relies on `let_chains`), organized as a Cargo workspace (`resolver = "2"`) with many crates (agent, app-server, cli, config, execpolicy, hooks, lane, mcp, protocol, secrets, state, tools, tui, workflow, workflow-js).
- Async runtime and networking: tokio 1.50 (multi-thread), axum 0.8.5, reqwest 0.13.1 with rustls (no default provider, socks), rustls 0.23.
- Storage and config: rusqlite 0.39 (bundled SQLite), toml / toml_edit, serde / serde_json, jsonschema 0.47.
- CLI and workflow: clap 4.5 with clap_complete; rquickjs 0.12 (a single-threaded QuickJS VM bridged to the engine over channels) for the Workflow VM.
- Observability and misc: tracing / tracing-subscriber / tracing-appender, mimalloc allocator; release profile uses `lto = true`, `codegen-units = 1`, and deliberately keeps unwinding (no `panic = "abort"`) so a panicking tool call fails gracefully.
- Distribution: npm workspace (`codewhale`, `deepseek-tui`, `runtime-sdk`; devDependency wrangler) and crates.io (`codewhale-cli`), version 0.9.1.
- MCP support via a dedicated `crates/mcp`.

## When to reach for it

- You want a single terminal coding agent that works across many hosted and local providers without wiring a separate key per provider.
- You need to switch models mid-task or route work to open models first.
- You want read-only planning, explicit approval gates, and OS-level sandboxing before an agent touches your machine.
- You want durable, resumable agent runs with an append-only ledger and per-turn receipts, including in CI via a headless `exec` mode.
- You want to enforce repo-level write policy through a `constitution.json` that even full-access runs cannot bypass.

## When *not* to reach for it

- You want a graphical IDE experience rather than a terminal TUI plus optional loopback browser client.
- You are committed to a single vendor's tightly integrated assistant and do not need multi-provider routing or model swapping.
- You cannot install a recent Rust toolchain (1.88+) when building from source, or your platform lacks the sandbox primitives the safety model expects.
- You need a mature, long-stable release line; the project is at an early 0.x version and moving quickly.

## Maturity signal

The repository was created in January 2026 and is pushed to almost daily, so it is under active, fast-moving development. Its large star and fork counts were gathered in only a few months, indicating strong early adoption rather than a long-settled tool, and the several hundred open issues fit a young project with a growing contributor base. The workspace version is 0.9.1 — pre-1.0 — and the MIT license plus the "independent community project, not affiliated with any model provider" framing signal an open, contribution-driven effort. Read it as actively maintained and popular, but still stabilizing toward a 1.0.

## Alternatives

- Aider — reach for it when you prefer a Python-based terminal assistant with tight git-commit integration over a Rust runtime.
- OpenAI Codex CLI — use when you want a vendor-maintained terminal agent and are comfortable centering it on one provider's models.
- Claude Code — use when you want a single-vendor, supported coding agent rather than a community project that puts open models first and swaps providers freely.

## Notes

- Pricing and context budgets are derived from the actual resolved route, and the project is explicit that an unknown price surfaces as "unknown" instead of being silently rendered as $0.
- Migration is a first-class concern: users coming from `deepseek-tui` are told their config and sessions carry over, and the repo keeps a `docs/REBRAND.md`. The README ships in seven languages.

## Tags

rust, cli, tui, coding-agent, llm, terminal, sandboxing, mcp, developer-tools
