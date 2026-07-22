# winfunc/opcode

A desktop GUI and toolkit for Claude Code that manages sessions, custom agents, usage, and checkpoints in a visual application.

## What it is

opcode is a Tauri 2 desktop application that provides a graphical interface for Anthropic's Claude Code CLI. It targets developers who already use Claude Code from the terminal and want a visual layer for browsing and resuming sessions, building custom agents, tracking API usage, and checkpointing work. According to the README, all data stays on the local machine and agents run in isolated processes.

## Key features

- Visual project and session browser over `~/.claude/projects/`, with search and the ability to resume past sessions with context.
- CC Agents: custom agents defined by system prompt and model, with per-agent file and network permissions, run in separate background processes and logged in an execution history.
- Usage analytics dashboard tracking cost and tokens broken down by model, project, and time period, with data export.
- MCP server management: add servers via UI or JSON, test connectivity, and import configurations from Claude Desktop.
- Timeline and checkpoints: session versioning, one-click restore, forking from a checkpoint, and a diff viewer.
- Built-in CLAUDE.md editor with live markdown preview and a project-wide scanner.

## Tech stack

- Frontend: React 18 with TypeScript (~5.6) and Vite 6.
- Backend: Rust (1.70.0 or later) with Tauri 2.
- UI: Tailwind CSS v4 and shadcn/ui, built on Radix UI primitives, with framer-motion and lucide-react.
- Database: SQLite via rusqlite.
- Package manager and runtime: Bun.
- State and forms: Zustand, react-hook-form, Zod.
- License AGPL-3.0; `package.json` version 0.2.1.

## When to reach for it

- You use Claude Code and want a GUI to browse, search, and resume sessions instead of the terminal.
- You want to build reusable custom agents with scoped file and network permissions.
- You want to track Claude API cost and token usage visually and export it.
- You manage several MCP servers and want a central place to configure and test them.
- You want session checkpoints, forking, and diffs to navigate a coding session's history.

## When *not* to reach for it

- You do not use Claude Code — opcode is a companion to that CLI, not a standalone coding tool.
- You want prebuilt installers today — the README states release executables "will be published soon" and directs users to build from source with the Rust and Bun toolchain.
- AGPL-3.0 is incompatible with your closed-source or commercial redistribution requirements.
- You prefer a terminal-only workflow and do not want a separate desktop application.

## Maturity signal

opcode was created in mid-2025 and drew roughly 22,000 stars within a few months, with the last push in October 2025 and no archive flag, so it is popular and actively developed. That said, the `package.json` version is 0.2.1 and the README still lists release binaries as forthcoming, which places it firmly in early, pre-1.0 territory rather than a settled release. The 333 open issues are consistent with rapid, high-traffic development.

## Alternatives

- Claude Code (Anthropic's CLI) — use it directly when you prefer the terminal and do not need the GUI or analytics layer.
- Cursor — use it when you want an AI-first editor rather than a session manager wrapped around Claude Code.
- Cline (VS Code extension) — use it when you want an agentic coding assistant embedded in VS Code instead of a separate desktop app.

## Notes

- The README's clone URL and acknowledgments reference `getAsterisk/opcode` and the Asterisk team, while the repository itself is `winfunc/opcode`, suggesting a fork or rename.
- The Security section claims "No Telemetry: No data collection or tracking," yet `package.json` includes `posthog-js` and the source tree contains an `AnalyticsConsent` component, so analytics appear to be present behind a consent flow.

## Tags

typescript, rust, tauri, desktop-app, gui, claude-code, llm, developer-tools, sqlite, react
