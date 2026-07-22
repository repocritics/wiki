# can1357/oh-my-pi

A terminal AI coding agent that wires IDE-grade tooling — hash-anchored edits, LSP, a debugger, browser, and subagents — directly into the model.

## What it is

oh-my-pi (the `omp` CLI) is a command-line and TUI coding agent, described as a fork of Mario Zechner's Pi. It gives the model a large built-in tool surface backed by a Rust native core, so operations like search, shell, AST rewrites, and file reads run in-process rather than by shelling out. It targets developers who want a batteries-included agent that works across many model providers and runs on macOS, Linux, and Windows.

## Key features

- 32 built-in tools in one namespace, including `read`/`write`/`edit`, `ast_edit`/`ast_grep`, `search`/`find`, `bash`, `eval` (persistent Python and JavaScript), `lsp`, `debug`, `task`, `browser`, and `web_search`.
- Hashline editing: the model anchors edits by content hash instead of retyping lines, and stale anchors cause the patch to be rejected before it applies.
- LSP wired into writes, including rename through `workspace/willRenameFiles`, plus a real DAP debugger driving lldb, dlv, and debugpy.
- Roughly 55,000 lines of Rust across crates (`pi-natives`, `pi-shell`, `pi-ast`, `pi-iso`) providing in-process grep, shell, AST, PTY, and more.
- 40+ model providers and a 25-backend `web_search` tool, with roles (`default`, `smol`, `slow`, `plan`, `commit`) that route work by intent.
- First-class subagents via `task` that return schema-validated results and can run in isolated worktrees.
- Four entry points: interactive TUI, one-shot (`omp -p`), RPC over stdio, and ACP for editor use (for example, inside Zed); a Node SDK is published as `@oh-my-pi/pi-coding-agent`.

## Tech stack

- TypeScript running on Bun (bun ≥ 1.3.14; `packageManager` is `bun@1.3.14`).
- Rust core, edition 2024, workspace version 17.0.7, MIT-licensed, exposed as an N-API addon for `linux-x64/arm64`, `darwin-x64/arm64`, and `win32-x64`.
- Vendored brush-shell for the embedded bash; ast-grep and tree-sitter for structural edits; ripgrep, syntect, and tiktoken-rs among the native dependencies.
- Puppeteer-core (patched) for the browser tool; OpenTelemetry for instrumentation; arktype and zod for schemas.
- Distributed via npm, an install script (`omp.sh/install`), Homebrew, and mise.

## When to reach for it

- You want a terminal coding agent with IDE-grade tooling (LSP, debugger) rather than plain shell-out commands.
- You switch between many model providers or coding-plan subscriptions and want a single harness.
- You want subagents, browser automation, or debugger-driven workflows from the command line.
- You want to embed an agent through the Node SDK, RPC, or ACP inside an editor.

## When *not* to reach for it

- You want a small, stable tool — this is fast-moving (created December 2025) with a wide surface and 889 open issues.
- Your platform cannot run Bun or the prebuilt native addon.
- You specifically want the minimal upstream Pi experience — omp is the batteries-included fork that adds the extra tooling.
- You prefer a hosted product or a narrow, vendor-supported CLI over a self-run agent.

## Maturity signal

Created at the end of December 2025, omp reached roughly 19,000 stars within about seven months and was last pushed in July 2026, indicating very active and rapid iteration. The 889 open issues reflect both heavy real-world use and a broad feature surface that is still stabilizing. It is MIT-licensed, published to npm, and has CI; the Rust workspace version of 17.0.7 signals a fast release cadence rather than a long-term stability guarantee.

## Alternatives

- Pi (badlogic/pi-mono) — use the upstream project when you want the smaller, original agent without the added tooling.
- Claude Code or a similar first-party CLI — use it when you want a vendor-supported agent with a narrower tool set.
- Aider — use it when you want a mature, git-centric terminal coding assistant.

## Notes

- It is explicitly a fork of Pi, adding tooling on top of the original.
- Several advanced tools (`github`, `tts`, `checkpoint`, `rewind`, `retain`, `recall`, `reflect`, `inspect_image`) are setting-gated and off by default.
- It reads eight existing agent-config formats natively (Cursor MDC, Cline `.clinerules`, Codex `AGENTS.md`, Copilot `applyTo`, and others) with no migration step.
- GitHub PRs and issues, merge conflicts, and subagent outputs are addressed as filesystem-style paths (`pr://`, `conflict://`, `agent://`), so the same `read`/`search` tools work on them.
- Three lowercase prose keywords (`ultrathink`, `orchestrate`, `workflowz`) opt a turn into specialized behavior, but only when they appear in prose, not in code spans or identifiers.

## Tags

typescript, rust, cli, tui, coding-agent, llm, bun, developer-tools, lsp, ast, terminal
