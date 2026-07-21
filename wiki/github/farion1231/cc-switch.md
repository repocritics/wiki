# farion1231/cc-switch

A cross-platform desktop app that manages provider configs, MCP servers, and skills for several AI coding tools from one interface.

## What it is

CC Switch is a desktop application that centralizes configuration for AI coding tools that each keep their own config format — Claude Code, Claude Desktop, Codex, Gemini CLI, Grok Build, OpenCode, OpenClaw, and Hermes Agent. Instead of hand-editing JSON, TOML, or `.env` files to change API providers, you switch providers from a visual interface or the system tray. It is aimed at developers who juggle multiple agentic coding tools and multiple API providers and want one place to manage them.

## Key features

- One interface to manage the supported tools, with 50+ built-in provider presets (the README names AWS Bedrock, NVIDIA NIM, and community relays) so importing a provider is a pick-and-switch rather than a file edit.
- Unified MCP and Skills management across the supported tools, with bidirectional sync.
- System tray quick switching, so a provider change does not require opening the full app.
- Cloud sync of provider data across devices via Dropbox, OneDrive, iCloud, or WebDAV.
- Configs backed by a SQLite database with atomic writes to protect against corruption.
- Built-in utilities for first-launch login confirmation, signature bypass, and plugin extension sync.

## Tech stack

- Rust backend built on Tauri 2 (`@tauri-apps/cli` and `@tauri-apps/api` ^2.8.0), packaged as a native desktop app for Windows, macOS, and Linux.
- React 18 + TypeScript 5.3 frontend, bundled with Vite 7, styled with Tailwind CSS 3.4 and Radix UI primitives.
- TanStack Query 5 for data, react-hook-form + zod 4 for forms/validation, i18next / react-i18next for localization, CodeMirror 6 for config editing, and recharts for charts.
- Tauri plugins for dialog, log, process, store, and updater; project version 3.17.0 in `package.json`; managed with pnpm.

## When to reach for it

- You use more than one of the supported AI coding tools and switch API providers often.
- You want to avoid hand-editing each tool's JSON/TOML/`.env` config.
- You want one panel to manage MCP servers and skills across tools rather than configuring each separately.
- You want provider settings synced across machines through an existing cloud storage service.

## When *not* to reach for it

- You only use a single tool and rarely change providers — that tool's own settings are enough.
- You need a headless or scriptable/CLI workflow; this is a GUI desktop app.
- You want in-request routing between models inside one tool rather than switching whole-tool configurations (see Alternatives).
- You are on a platform outside Windows, macOS, or Linux.

## Maturity signal

The project is actively and heavily maintained: created in August 2025, pushed the same day this page's facts were captured (July 2026), with a very high star count, a steady stream of versioned release notes in `docs/`, and an MIT license. The open-issue count is large (over two thousand), which for a popular desktop tool reads as high user volume and a broad support surface rather than neglect. The Tauri 2 base and release cadence point to ongoing development.

## Alternatives

- **Editing each tool's config files directly** — use this when you only touch one provider or one tool and do not need presets, sync, or a UI.
- **[claude-code-router](https://github.com/musistudio/claude-code-router)** — use a request-routing proxy when you want to route model traffic within a single tool (e.g. Claude Code) to different backends, rather than switching a tool's whole configuration.
- **A tool's own built-in provider settings** — use these when the tool already exposes provider switching and you do not need cross-tool unification.

## Notes

- The README is dominated by a long list of sponsor/API-relay advertisements; the actual product description ("Why CC Switch?", features, screenshots) sits below that block.
- The README stresses that `ccswitch.io` is the only official website, a signal that lookalike or impostor distributions are a concern for this project.

## Tags

rust, typescript, tauri, react, desktop-app, gui, developer-tools, ai-tools, claude-code, configuration-management, mcp
