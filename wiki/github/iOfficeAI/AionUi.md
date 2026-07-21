# iOfficeAI/AionUi

A free, open-source desktop application that runs AI agents locally and unifies many command-line coding agents behind one Cowork interface.

## What it is

AionUi is a cross-platform Electron desktop app that turns AI agents into a graphical Cowork workspace where agents can read and write files, browse the web, run code, and produce documents on your machine. It ships a built-in agent that works with any API key, and it also auto-detects and drives external CLI agents such as Claude Code, Codex, and Gemini CLI through a single interface. It targets users who want a local, model-agnostic assistant that can act on their files and run unattended.

## Key features

- A built-in agent engine that works immediately with any provider API key, bundling 21 professional assistants (Cowork, PPT/Word/Excel creators, academic paper writer, and more).
- Multi-agent mode that auto-detects installed CLI agents (Claude Code, Codex, Qwen Code, OpenCode, and others) and runs them in a unified interface with parallel sessions.
- Team mode where a Leader agent delegates subtasks to Teammate agents that execute in parallel and share a task board via a built-in Team MCP server.
- Remote access through a WebUI mode and chat-platform integrations (Telegram, Lark, DingTalk, WeChat).
- Scheduled tasks via cron expressions, fixed intervals, or one-time triggers for unattended runs.
- A preview panel for 10+ output formats (PDF, Office documents, code, Markdown, images, diffs) with editing and Git-based version history, plus Office document generation via the companion OfficeCLI.

## Tech stack

- TypeScript on Electron, built and run with electron-vite; current version 2.1.39.
- React 19 with the Arco Design component library, CodeMirror and Monaco editors, and i18next for localization.
- Local storage in SQLite via `better-sqlite3`.
- Provider SDKs including `@anthropic-ai/sdk`, `@google/genai`, `openai`, `@aws-sdk/client-bedrock`, and the `@modelcontextprotocol/sdk`; agent integration via `@agentclientprotocol/sdk`.
- Chat and web integrations using grammy (Telegram), the Lark/Feishu SDK, Express, and WebSockets; scheduling via croner.
- Packaged for macOS, Windows, and Linux with electron-builder; workspaces managed with Bun.

## When to reach for it

- Wanting a local desktop assistant that can operate on your own files and run multi-step tasks with your approval.
- Already using CLI agents (Claude Code, Codex, Gemini CLI) and wanting one GUI that detects and runs them together.
- Needing scheduled, unattended automation or remote control of an agent from a phone via WebUI or chat apps.
- Generating editable Office documents (PPTX, DOCX, XLSX) from natural-language requests.

## When *not* to reach for it

- Users who want a purely hosted, zero-install web service; AionUi is a desktop app you run yourself.
- Environments where granting an agent full local file access and auto-approve (YOLO/full-auto) modes is unacceptable.
- Anyone needing a single vetted model with vendor support rather than a broad, community-driven multi-agent aggregator.
- Machines below the recommended 4GB memory and 500MB storage.

## Maturity signal

Created in August 2025, AionUi has climbed past 30,000 stars with commits pushed within the last day, reflecting fast growth and active development rather than a settled product. The large open-issue count (nearly 700) is consistent with a young project moving quickly and absorbing heavy user interest. It is Apache-2.0 licensed with frequent releases (version 2.1.39) and multi-language documentation.

## Alternatives

- Cherry Studio — use instead when you want a desktop multi-provider chat client without CLI-agent orchestration or unattended automation.
- LibreChat or Open WebUI — use instead when you want a self-hosted, browser-based multi-user chat front end rather than a local desktop cowork app.
- Claude Cowork — use instead when you are macOS-only, want a first-party Anthropic app, and are content with Claude as the sole model (per the README's own comparison).
- Running the CLI agents directly (Claude Code, Codex, Gemini CLI) — use instead when you prefer the terminal and do not need a GUI or aggregation.

## Notes

AionUi positions itself explicitly against Claude Cowork in the README, framing itself as a cross-platform, any-model version. All data is stored locally in a SQLite database with nothing uploaded to a server, and MCP configuration is set once and synced automatically to every connected agent rather than configured per agent.

## Tags

typescript, electron, desktop-app, ai-agent, llm, mcp, chatbot, cross-platform, automation, react
