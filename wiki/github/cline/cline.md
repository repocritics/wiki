# cline/cline

An autonomous coding agent distributed as an SDK, VS Code extension, or CLI assistant — runs LLM-driven coding workflows inside a developer's IDE.

## What it is

A TypeScript-based autonomous coding agent originally known as Claude Dev. Operates inside VS Code (and via a separate SDK / CLI) as an agent that can plan, edit files, run terminal commands, and iterate on tasks. Multi-provider — works against Anthropic Claude, OpenAI GPT-4 family, and other LLM endpoints. Apache 2.0 licensed.

## Key features

- VS Code extension as the primary surface; SDK + CLI as secondary distribution.
- Multi-provider: Claude, OpenAI, OpenRouter, Bedrock, Vertex, local LLMs.
- Tool-using agent loop with file edit, terminal, MCP tool integration.
- Plan + Act mode separation for explicit user review.
- Approval flow before file edits / terminal commands.
- Cost tracking for API spend per session.
- Apache 2.0 licensed.

## Tech stack

- TypeScript primary.
- VS Code Extension API.
- MCP client integration.

## When to reach for it

- You want a multi-provider agent inside VS Code (rather than vendor-specific like GitHub Copilot Chat).
- You want explicit plan/act separation for safer auto-edits.
- You're comparing the OSS coding-agent landscape and want a Apache-2.0-licensed reference.

## When *not* to reach for it

- You want fully vendor-supported tooling — Claude Code, GitHub Copilot, Cursor are first-party alternatives.
- You're allergic to OSS-with-paid-LLM-cost — the agent needs API keys from a paid provider.

## Maturity signal

63k stars, 6.6k forks, Apache 2.0, actively maintained. ~2 years old; renamed from "Claude Dev" to "cline" as the project broadened to multi-provider support.

## Alternatives

- `anthropics/claude-code` — first-party Anthropic equivalent.
- GitHub Copilot Chat agent mode — first-party GitHub.
- Cursor, Windsurf — IDE-flavored alternatives.

## Tags

artificial-intelligence, large-language-model, agent, typescript, vscode, claude, openai, coding-agent, apache-license, model-context-protocol
