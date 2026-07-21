# bytedance/UI-TARS-desktop

An open-source multimodal AI agent stack that drives computers and browsers through natural language, shipping both a desktop GUI agent and a CLI-based agent.

## What it is

UI-TARS-desktop is a repository that ships two related projects under one multimodal AI agent stack: Agent TARS and UI-TARS Desktop. Agent TARS is a general multimodal agent that runs from a terminal CLI or Web UI and integrates with real-world tools over MCP, combining GUI-agent and vision capabilities. UI-TARS Desktop is a native desktop application that operates the local computer and browser as a GUI agent, driven by the UI-TARS and Seed-1.5-VL/1.6 models. It is aimed at developers who want agents that complete tasks by seeing and controlling a real screen rather than only calling APIs.

## Key features

- Natural-language control of a computer powered by a Vision-Language Model, with screenshot capture and visual recognition, and precise mouse and keyboard control.
- Local and remote operators for both computer and browser use; the remote computer and browser operators are described as free and require no configuration.
- Agent TARS ships a one-click CLI supporting both a headful Web UI and headless server execution.
- Hybrid browser agent that can control browsers via GUI Agent (visual grounding), DOM, or a hybrid strategy.
- MCP-based kernel that also supports mounting external MCP servers to connect to real-world tools.
- Cross-platform desktop support (Windows, macOS, Browser) with real-time feedback and status display, running fully local processing.

## Tech stack

- TypeScript, organized as a pnpm monorepo (`pnpm@9.10.0`) using Turbo (`turbo ^2.4.4`).
- Electron desktop application (Electron Forge, electron-vite, electron-builder configs present).
- Node.js `>=20.x` at the monorepo level; the Agent TARS CLI requires Node.js `>=22`.
- Testing with Vitest and Playwright; linting/formatting via ESLint and Prettier; Changesets for release management.
- Distributed on npm as `@agent-tars/cli`; the UI-TARS SDK is provided as a cross-platform toolkit for building GUI automation agents.

## When to reach for it

- You need an agent that operates real GUI applications and browsers via vision rather than through structured APIs.
- You want a desktop app that can control your local machine with natural-language instructions.
- You want to run browser or computer automation tasks against a remote operator without local setup.
- You are building GUI automation on top of the UI-TARS model family and want a CLI, Web UI, or SDK entry point.

## When *not* to reach for it

- Your automation targets have stable APIs or DOM selectors; a vision-driven GUI agent adds cost and latency you may not need.
- You need a small dependency-light library; this is a large Electron monorepo plus model integration.
- You cannot supply a capable multimodal model endpoint (for example Volcengine, Anthropic, or a self-hosted UI-TARS deployment), which the agent depends on.
- You require a mature, low-churn tool; the project is young and moves quickly.

## Maturity signal

The repository has over 38,000 stars accumulated since its creation in January 2025, and its most recent push was July 2026, making it a fast-moving, actively maintained project backed by ByteDance and licensed under Apache-2.0. The large open-issue count (over 400) reflects heavy real-world use and rapid iteration rather than abandonment, and the frequent versioned releases noted in the README (UI-TARS Desktop v0.1.0 through v0.2.0, Agent TARS CLI v0.3.0) reinforce an early-but-active trajectory. Treat APIs and features as still evolving.

## Alternatives

- browser-use — use it instead when you only need browser automation driven by an LLM and do not need native desktop control.
- Midscene — use it instead when you want in-browser GUI automation; the README itself links it as the browser-focused counterpart.
- Self-Operating Computer — use it instead when you want a lightweight, model-agnostic script for computer control rather than a packaged desktop app.
- Anthropic Claude computer use — use it instead when you are already in the Claude ecosystem and want a hosted computer-use capability.

## Notes

The repository is a stack rather than a single app: Agent TARS (CLI/Web UI, MCP-centric) and UI-TARS Desktop (Electron desktop GUI agent) are shipped together but serve different usage modes. Agent TARS's kernel is built directly on MCP, and it uses a protocol-driven Event Stream to power both its context engineering and its Agent UI. The v0.3.0 release adds an AIO Agent Sandbox as an isolated all-in-one tool-execution environment.

## Tags

typescript, electron, ai-agent, gui-automation, computer-use, browser-automation, multimodal, vlm, mcp, cli, llm
