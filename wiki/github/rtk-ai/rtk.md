# rtk-ai/rtk

A single-binary Rust CLI proxy that filters and compresses command output before it reaches an LLM, cutting token use by a reported 60-90%.

## What it is

rtk sits between an AI coding agent and the shell, intercepting common development commands and compressing their output before it enters the model's context. It targets developers running agentic coding tools that shell out frequently and spend tokens on verbose command output. It ships as a single Rust binary with no runtime dependencies and low per-command overhead.

## Key features

- Filters and compresses output for 100+ commands across files, git, GitHub CLI, test runners, build and lint tools, package managers, AWS, containers, infrastructure-as-code, and data utilities, with a stated overhead under 10ms.
- An auto-rewrite hook that transparently rewrites Bash commands (for example `git status` to `rtk git status`) before execution, across 15 supported AI tools including Claude Code, Copilot, Cursor, Gemini CLI, Codex, Windsurf, Cline, OpenCode, Pi, and Hermes.
- Four compression strategies applied per command type: smart filtering, grouping, truncation, and deduplication.
- Token-savings analytics (`rtk gain`, `rtk discover`, `rtk session`) with summary stats, graphs, history, and JSON export.
- Tee recovery that saves full unfiltered output on failure so the model can read it without re-running the command.
- Opt-in, anonymous, aggregate telemetry that is disabled by default and requires explicit consent.

## Tech stack

- Rust (edition 2021, `rust-version` 1.91), packaged as a single binary; the release profile uses LTO, `panic = "abort"`, and stripping.
- clap 4 for the CLI, anyhow for errors, regex, and serde/serde_json (with preserved key order).
- rusqlite (bundled SQLite) for analytics storage, toml for config, chrono for time handling, ureq for HTTP, quick-xml, and sha2 for the salted device hash used by telemetry.
- `unsafe_code` and compiler warnings are denied in the lint configuration.
- Distributed via Homebrew, `cargo install`, an install script, and prebuilt binaries for macOS, Linux, and Windows; a native binary hook works on Windows without a Unix shell.

## When to reach for it

- You run an AI coding agent that executes many shell commands and want to reduce token consumption on their output.
- You want transparent savings via a hook, without changing how you invoke commands.
- You use one of the supported agents and want per-command analytics on tokens saved.

## When *not* to reach for it

- Your agent relies on built-in Read, Grep, or Glob tools; those bypass the Bash hook and are not rewritten.
- You rarely route shell commands through the model, so there is little output to compress.
- You need output verbatim and cannot tolerate any filtering, though tee recovery mitigates this on failures.
- The commands you run are outside the supported set, where rtk falls back to passthrough.

## Maturity signal

The project has drawn a very large star count within roughly six months of creation and was pushed on the snapshot date, indicating very active development and rapid adoption. It is Apache-2.0 licensed and not archived, with a `develop` default branch, a named core team, and a high open-issue count alongside frequent version bumps — all consistent with a fast-moving, actively maintained tool that is still stabilizing.

## Alternatives

- repomix — use it instead when you want to pack a whole repository into one compact file for an LLM, rather than filter live command output.
- code2prompt — use it instead when your goal is turning a codebase into a single token-efficient prompt rather than proxying individual shell commands.

## Notes

There are two version signals worth flagging: the README's verification example shows `rtk 0.28.2`, while the `Cargo.toml` declares `0.42.4`, so the documented version lags the manifest. The README also warns of a name collision with an unrelated crates.io package called "rtk" (Rust Type Kit), recommending a git-based install if the wrong one is picked up.

## Tags

rust, cli, llm, developer-tools, token-optimization, ai-coding, claude-code, command-line-tool, proxy
