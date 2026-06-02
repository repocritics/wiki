# google-gemini/gemini-cli

Google's official open-source AI agent CLI — brings Gemini into the terminal with tool use, MCP support, and an agentic loop.

## What it is

A TypeScript-based command-line agent from Google that wraps the Gemini API (Pro / Flash / Nano models) with a Claude-Code-style harness: tool calls, file edits, terminal access, session memory, MCP client + server support. Distributed at geminicli.com, installable via npm. Apache 2.0 licensed. The Google-first equivalent of Anthropic's Claude Code.

## Key features

- Gemini-powered agent loop (model-callable tools: file edit, shell, search, etc.).
- MCP (Model Context Protocol) client + server.
- Apache 2.0 licensed — first-party open-source.
- Cross-platform (Linux, macOS, Windows).
- Multi-model support across Gemini family (Pro, Flash, Ultra variants).
- Tight integration with Google Cloud (Vertex AI) for enterprise auth.

## Tech stack

- TypeScript primary.
- Node.js runtime, npm distribution.

## When to reach for it

- You're Gemini-aligned and want a first-party CLI agent.
- You're comparing first-party offerings: Claude Code (Anthropic), Codex (OpenAI), gemini-cli (Google).
- You want an Apache-2.0-licensed reference agent.

## When *not* to reach for it

- You're allergic to Google-product lock-in.
- You want a multi-provider agent — `cline/cline` or third-party harnesses fit better.

## Maturity signal

105k stars, 14k forks, Apache 2.0, actively maintained. Open-issues count of 1,357 tracks the breadth of Gemini's evolving API + MCP-integration requests.

## Alternatives

- `anthropics/claude-code` — Anthropic equivalent.
- OpenAI Codex CLI.
- `cline/cline`, `obra/superpowers` — third-party.

## Tags

artificial-intelligence, large-language-model, agent, typescript, command-line-interface, gemini, google, model-context-protocol, apache-license, coding-agent
