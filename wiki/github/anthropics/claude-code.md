# anthropics/claude-code

Anthropic's official agentic coding tool — a CLI that lives in the terminal, understands a codebase, and executes routine engineering tasks through natural-language commands.

## What it is

A terminal-based coding agent built by Anthropic that wraps Claude (the company's LLM family) into a workflow-aware harness: it can read files, run commands, edit code, navigate git history, and complete multi-step engineering tasks while the user supervises. Distinguishes itself from chat-only LLM interfaces by owning a session loop with tool calls, persistent context, and a skills system that lets it specialize per project. Closed-source binary; this repository hosts public-facing assets (docs, issue tracker, example skills/configs) rather than the agent's core code.

## Key features

- Tool-using agent loop — Bash, Read, Edit, Write, Grep, file-search, and many more tools available to the model per turn.
- Session memory + plan tracking — keeps state across turns within a working session.
- Per-project conventions via CLAUDE.md / AGENTS.md files in the working tree.
- Skills system (paired with `anthropics/skills`) — agents can install and invoke task-specific recipes.
- MCP (Model Context Protocol) client and server support — integrates with the broader tool ecosystem.
- Background execution + notification hooks for long-running operations.
- Multi-model fallback — Sonnet by default, with Opus for heavier reasoning and Haiku for fast lookups.

## Tech stack

- Python primary at the public repo level (issue/PR scaffolding, example scripts).
- The agent runtime ships as a closed-source CLI binary distributed via npm / Homebrew / direct install.
- Skills layer is documented and partially open via `anthropics/skills`.

## When to reach for it

- You want a coding agent that's first-party Anthropic-supported and tightly integrated with Claude's tool-use capabilities.
- You want a CLI / IDE-adjacent agent rather than a chat-only interface.
- You're building per-project conventions (CLAUDE.md) and want an agent that respects them out of the box.

## When *not* to reach for it

- You want a fully open-source agent — the runtime is closed-source; only docs and skill bundles are open here.
- You need provider-flexibility — Claude Code is Claude-only; OpenAI Codex CLI is the parallel choice for GPT models.
- You're allergic to vendor lock-in for your dev workflow — switching agent vendors mid-project is friction.

## Maturity signal

129k stars accumulated since February 2025 — fast-rising, but proportional to the broader "coding agent" wave that Anthropic both rode and helped accelerate. Last push the morning this page was generated. The 8,369 open-issues count is high; agent harnesses surface many feature requests and edge-case reports given the breadth of tasks users attempt. License absence here is the recurring gotcha — the agent binary is proprietary, the public repo's content licensing varies.

## Alternatives

- OpenAI Codex CLI — first-party OpenAI equivalent for GPT models.
- Cursor, Windsurf — IDE-integrated agent experiences (not CLI-first).
- `obra/superpowers`, `affaan-m/ECC` — third-party agent / skills frameworks that target Claude Code (and others) as the host harness.

## Notes

The public repo serves as the issue tracker, documentation surface, and skills/conventions reference — not the source of the agent itself. Anyone re-indexing GitHub for "open-source coding agents" should weight this differently than fully-OSS competitors. The skills + CLAUDE.md + MCP integration matrix is the most-distinctive piece of the offering; the model intelligence is shared with the broader Claude API.

## Tags

artificial-intelligence, large-language-model, agent, claude, anthropic, command-line-interface, coding-agent, developer-tools, model-context-protocol
