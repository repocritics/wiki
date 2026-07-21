# Yeachan-Heo/oh-my-claudecode

A team-first multi-agent orchestration layer for Claude Code that adds autonomous execution modes, tmux-based CLI workers, and cross-provider advisors on top of the base tool.

## What it is

oh-my-claudecode (OMC) is a multi-agent orchestration system for Claude Code that aims to let users describe what they want in natural language instead of learning Claude Code's mechanics. It exposes two surfaces: in-session slash commands and prompt triggers, and a terminal CLI (`omc`) that can spawn real worker processes in tmux panes. Its canonical orchestration mode is Team, a staged pipeline that plans, writes a PRD, executes, verifies, and loops on fixes. It targets developers who want coordinated, persistent, cost-aware agent workflows and optional cross-checking from other model CLIs.

## Key features

- Team mode as the canonical surface, running a staged pipeline (`team-plan → team-prd → team-exec → team-verify → team-fix`) with an in-session native variant and a tmux-CLI variant.
- tmux CLI workers that launch real `claude`, `codex`, `gemini`, `antigravity`, `grok`, or `cursor-agent` processes on demand and shut them down when their task completes.
- Multiple execution modes including Autopilot (autonomous single lead), Ultrawork (maximum parallelism), Ralph (persistent verify/fix loops), UltraQA (quality-gate cycling), and Pipeline.
- 19 specialized agents with tier variants and smart model routing between Haiku and Opus for cost optimization.
- A provider advisor (`omc ask` / `/ask`) that runs local provider CLIs and saves markdown artifacts, plus a `/ccg` tri-model synthesis flow.
- Custom, auto-injecting skills extracted from sessions, a HUD statusline, session/replay artifacts, and configurable notification callbacks for Telegram, Discord, and Slack.

## Tech stack

- TypeScript, targeting Node.js `>=20.0.0`, built with `tsc` and `esbuild`.
- Depends on `@anthropic-ai/claude-agent-sdk ^0.1.0` and `@modelcontextprotocol/sdk ^1.26.0`.
- Uses `better-sqlite3` for local state, `commander` for the CLI, `zod`/`ajv` for validation, `@ast-grep/napi` for structural code operations, and `vscode-languageserver-protocol`.
- Tested with Vitest; distributed on npm as `oh-my-claude-sisyphus` (this excerpt is version 4.15.4), exposing both `oh-my-claudecode` and `omc` binaries.
- Also exports TypeScript library helpers such as `createOmcSession()` for local Node.js programs. MIT licensed.

## When to reach for it

- You want Claude Code to coordinate multiple specialized agents through a staged, verified pipeline rather than issuing one-shot prompts.
- You want autonomous or persistent execution that keeps working until a task is verified complete.
- You want to cross-check work with Codex, Gemini, Grok, Antigravity, or Cursor CLIs from within one workflow.
- You want built-in observability (HUD, session replays) and multi-repo workspace state management.

## When *not* to reach for it

- You do not use Claude Code; OMC is an orchestration layer for it, not a standalone agent.
- You want interactive slash-command workflows in CI; the README advises against relying on `/autopilot`, `/ralph`, or `/team` in headless runs and to use deterministic `omc` commands instead.
- You need the tmux CLI workers but cannot install or authenticate the underlying CLIs or run an active tmux session.
- You prefer a minimal, low-churn setup; OMC is a large, fast-moving system with frequent releases and deprecations.

## Maturity signal

Despite being created only in January 2026, the repository has nearly 38,000 stars and a most-recent push in July 2026 with just a single open issue, pointing to intense early adoption and very active, closely managed maintenance. The rapid version churn (this excerpt is v4.15.4) and repeated breaking changes documented in the README — removal of the `swarm` keyword in favor of Team, removal of the Codex/Gemini MCP servers in v4.4.0 — indicate a project still reshaping its core surfaces, so expect ongoing migration work. It is MIT licensed.

## Alternatives

- wshobson/agents — use it instead when you want a catalog of installable agents, skills, and commands rather than a runtime orchestration engine.
- oh-my-codex — from the same author, use it instead when your primary harness is OpenAI Codex CLI rather than Claude Code.
- Claude Code's native agent teams — use them instead when you want first-party team execution without an added orchestration layer.

## Notes

The project's branding and the published npm package name diverge: the repo, plugin, and commands are branded oh-my-claudecode, but the npm package is `oh-my-claude-sisyphus`, which installs both the `oh-my-claudecode` and `omc` commands. The README also documents a deliberate architectural shift away from MCP servers toward CLI-first tmux workers, and it explicitly cautions against claiming that Claude Code's `/goal` evaluator independently runs commands or reads files.

## Tags

typescript, claude-code, ai-agents, multi-agent, orchestration, cli, automation, developer-tools, mcp, llm
