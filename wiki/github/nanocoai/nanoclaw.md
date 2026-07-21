# nanocoai/nanoclaw

A lightweight personal AI assistant that runs each agent in its own Linux container and connects to your messaging apps.

## What it is

NanoClaw is a small TypeScript AI assistant that positions itself as an easier-to-understand, isolation-first alternative to OpenClaw. It runs agents inside per-session Linux containers with filesystem isolation rather than application-level permission checks, connects them to messaging channels, and gives them memory and scheduled jobs. It is built for individual users who want to fork a compact codebase and have Claude Code customize it to their exact needs instead of configuring a large generic framework.

## Key features

- Container isolation: each agent group runs in its own Docker container and can only see explicitly mounted directories, so bash access runs inside the container, not on the host.
- Multi-channel messaging installed on demand via `/add-<channel>` skills — WhatsApp, Telegram, Discord, Slack, Microsoft Teams, iMessage, Matrix, Google Chat, Webex, Linear, GitHub, WeChat, and email via Resend — with flexible per-channel isolation.
- Per-agent workspace: each group has its own `CLAUDE.md`, memory, container, and mounts, plus reusable agent templates stamped via `ncl groups create --template`.
- Scheduled recurring tasks (with optional script gates), web search and fetch, and a scripted install flow that hands off to Claude Code when a step needs judgment.
- Credential security through OneCLI's Agent Vault: agents never hold raw API keys, and outbound requests get credentials injected at the proxy with per-agent policies and rate limits.
- Runs on Anthropic's Claude Agent SDK by default, with drop-in alternative providers (`/add-codex`, `/add-opencode`, `/add-ollama-provider`) configurable per agent group.

## Tech stack

- TypeScript, Node.js `>=20`, pnpm (`pnpm@10.33.0`); root package version 2.1.53.
- SQLite via `better-sqlite3` 11.10.0 (two files per session, one writer each), `cron-parser` 5.5.0 for scheduling, and `@clack/prompts` for the interactive installer.
- OneCLI SDK (`@onecli-sh/sdk` 2.2.1) for credential injection and a `chat` dependency (4.29.0) for the messaging bridge.
- Agent-runner inside the container uses Bun and the Claude Agent SDK; Docker is the runtime (macOS, Linux, Windows via WSL2).
- Tooling: Vitest, ESLint (including `eslint-plugin-no-catch-all`), Prettier, tsx, Husky; MIT-licensed. Exposes an `ncl` CLI binary.

## When to reach for it

- Running a personal assistant that reaches you across several messaging apps while keeping each agent sandboxed.
- Wanting a codebase small enough to read and modify, customized through code changes rather than configuration files.
- Needing scheduled agent jobs (briefings, reviews, reminders) triggered from chat.
- Isolating agents and credentials so an agent can only touch what you explicitly mount and never holds raw keys.

## When *not* to reach for it

- You cannot or don't want to run Docker; container isolation is central and Docker is the default runtime.
- You prefer configuration files and a stable, feature-complete framework over forking and modifying source per user.
- You want a batteries-included assistant where new channels and providers ship in the trunk; here they live on separate branches and are installed per-fork via skills.
- You need a large multi-tenant deployment; the design is oriented around individual, bespoke installs.

## Maturity signal

NanoClaw is young (created early 2026) but moving fast, with commits pushed the same week as this writing, a large following, and releases already in the 2.x line. Its fork count is exceptionally high relative to stars, which fits the stated design: users are expected to fork and customize their own copy. The high open-issue count is consistent with rapid, active development and heavy real-world use rather than neglect. MIT licensing keeps forking and customization low-risk. Treat it as an actively developed, deliberately minimal project whose base is intended to stay small.

## Alternatives

- OpenClaw — use instead when you want the more full-featured, batteries-included assistant and are comfortable with a much larger codebase and application-level permissioning.
- Anthropic's Claude Agent SDK, used directly — build on it yourself when you want a bespoke agent and don't need NanoClaw's multi-channel routing, container isolation, and scheduling scaffolding.
- n8n — use instead when your need is app-to-app workflow automation rather than a conversational, memory-carrying assistant.

## Notes

- Contribution policy is unusual: only security fixes, bug fixes, and clear improvements are accepted to the base; new channels and providers must land on the `channels`/`providers` registry branches and are installed per-fork.
- The architecture uses two SQLite files per session (`inbound.db` and `outbound.db`), each with exactly one writer, to avoid IPC and cross-mount contention between the host and container.
- There are no configuration files by design — customization means asking Claude Code to change the code — and the uninstaller tags each checkout so it removes only what that copy created.

## Tags

typescript, ai-agents, ai-assistant, claude, docker, sandboxing, self-hosted, messaging-bot, sqlite, scheduled-tasks
