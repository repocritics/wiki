# shareAI-lab/learn-claude-code

An educational, from-scratch tutorial that builds a Claude Code-style agent harness one mechanism at a time on top of a single agent loop.

## What it is

Learn Claude Code teaches "harness engineering" — the code that gives a trained model an operational environment — by building a small Claude Code-like agent from zero. It argues that agency comes from the model while the harness supplies tools, knowledge, context management, and permissions, and it walks through each harness mechanism in order. It is aimed at developers who want to understand how coding-agent harnesses work internally rather than to consume a finished product.

## Key features

- 20 progressive lessons (`s01`–`s20`), each adding one harness mechanism on top of an unchanging agent loop.
- Coverage of tool use, permissions, hooks, TodoWrite planning, subagents, on-demand skill loading, context compaction, memory, runtime system-prompt assembly, error recovery, a persisted task system, background tasks, cron scheduling, agent teams and protocols, autonomous agents, worktree isolation, and MCP.
- A standalone runnable `code.py` in each chapter, so mechanisms can be run and studied in isolation.
- Trilingual chapter material (Chinese source plus English and Japanese translations) with SVG diagrams where needed.
- A legacy 12-lesson track and a web app kept during the transition to the newer 20-lesson track.

## Tech stack

- Python, using the Anthropic SDK (`anthropic>=0.25.0`), python-dotenv (`>=1.0.0`), and PyYAML (`>=6.0`).
- Requires an `ANTHROPIC_API_KEY`, configured via a `.env` file.
- The core is the standard Anthropic messages API tool loop (create message, append assistant content, execute `tool_use` blocks, loop until stop).
- The web platform is a separate Node app (`web/`, run with `npm run dev`) that currently renders the legacy track.

## When to reach for it

- You want to learn how coding-agent harnesses work by reading and running minimal implementations.
- You are building your own harness and want reference patterns for subagents, compaction, task graphs, or worktree isolation.
- You are teaching agent architecture and want a progressive, chapter-by-chapter path.

## When *not* to reach for it

- You want a production-ready agent to use directly; the README points to Kode CLI and the Kode SDK for that.
- You need a maintained framework with guarantees; several production mechanisms are intentionally simplified or omitted for teaching.
- You do not use the Anthropic API or want a provider-agnostic implementation; the code targets the anthropic SDK.

## Maturity signal

The project has a very large star count and was created about a year before the snapshot, with its most recent push roughly a month earlier — a pattern of sustained, active maintenance rather than abandonment. It is MIT-licensed, not archived, and carries a modest open-issue count. It is mid-transition from a 12-lesson to a 20-lesson track, so the legacy and current materials coexist, which is a normal state for an evolving teaching repository.

## Alternatives

- shareAI-lab/Kode-CLI — use it instead when you want a ready-to-use open-source coding agent CLI rather than to build one from scratch.
- shareAI-lab/claw0 — use it instead when you want the always-on assistant variant (heartbeat, cron, IM channels, memory) rather than the use-and-discard harness taught here.
- Anthropic's cookbook and "Building Effective Agents" guidance — use them instead when you want official reference patterns rather than a build-it-yourself tutorial.

## Notes

The repository's thesis is stated bluntly: agency is trained, not coded, so the reader's job is the harness, not the intelligence. Two caveats about the content: the legacy and current tracks do not always share chapter numbers, so the README warns against mixing them, and the JSONL mailbox protocol used for team coordination is described as a teaching implementation rather than a claim about any production internal.

## Tags

python, ai-agent, llm, claude-code, educational, tutorial, agent-harness, anthropic, learning
