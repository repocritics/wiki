# DietrichGebert/ponytail

A skill/plugin that pushes an AI coding agent to write the least code that solves the task, rather than over-building.

## What it is

Ponytail is an always-on ruleset and set of commands for AI coding agents that biases them toward minimal, necessary code. Before writing anything, the agent walks a decision ladder — does this need to exist, is it already in the codebase, does the standard library or platform or an installed dependency already do it — and stops at the first rung that holds. It is aimed at developers who use agentic coding tools and find that those agents tend to over-engineer (installing a dependency and writing a wrapper component where a native element would do).

## Key features

- An "over-engineering" ladder the agent runs after it understands the problem: YAGNI, reuse, stdlib, native platform feature, installed dependency, one line, then the minimum that works.
- Explicit carve-outs so the agent never drops trust-boundary validation, data-loss handling, security, or accessibility in the name of brevity.
- Commands for reviewing and auditing: `/ponytail-review` (over-engineering review of the current diff), `/ponytail-audit` (whole-repo), `/ponytail-debt` (ledger of deferred shortcuts), `/ponytail-gain` (benchmark scoreboard), and `/ponytail-help`.
- Intensity levels — `lite`, `full` (default), `ultra`, `off` — set per session with `/ponytail`, or by default via the `PONYTAIL_DEFAULT_MODE` env var or `~/.config/ponytail/config.json`.
- Adapters for roughly twenty agents (Claude Code, Codex, GitHub Copilot CLI, OpenCode, Gemini CLI, Qoder, Hermes, Devin CLI, OpenClaw, and others), plus instruction-only fallbacks via `AGENTS.md` and per-tool rule files.
- The active ruleset is also injected into subagents spawned via the Agent tool, scopeable with the `PONYTAIL_SUBAGENT_MATCHER` regex.

## Tech stack

- JavaScript; distributed on npm as `@dietrichgebert/ponytail` (version 4.8.4 in the manifest), MIT licensed.
- Ships as an OpenCode plugin (`main` points at `.opencode/plugins/ponytail.mjs`) with keywords `opencode-plugin`, `pi-package`, `skills`, `qoder`.
- Two small Node.js lifecycle hooks for Claude Code and Codex, so `node` must be on the non-interactive shell PATH for always-on activation; the skills still work without it.
- Bundled skills under `skills/`, a Pi extension, and an MCP server (`ponytail-mcp/`); tests run via `node --test`.

## When to reach for it

- Your agent reaches for a library or writes a bespoke component when a native platform feature or existing code would do.
- You want a consistent, portable "write only what the task needs" rule across several different coding agents rather than re-authoring prompts per tool.
- You want a review or audit pass that hands back a delete-list of over-engineered code.
- You want to track shortcuts the agent deferred so they do not silently accumulate.

## When *not* to reach for it

- You want the agent to be more verbose or exploratory — ponytail biases the opposite direction.
- The README notes a reasoning model that spends thinking tokens deliberating the ladder can end up slower and costlier (it cites GPT-5.5 going the other way on cost/latency), so the cost/speed benefits are not universal.
- Your host has no skill or rule-file support at all; the plugin needs at least an instruction-file mechanism to inject the ruleset.
- You are optimizing the agent's prose length rather than the amount of code it produces — that is a different problem (see Alternatives).

## Maturity signal

The repository is recent (created mid-June 2026) but heavily developed, with a high star count, frequent pushes as of mid-July 2026, an MIT license, and a versioned npm release at 4.8.4. The breadth of adapters, a test suite, and generated-artifact checks point to active, fast-moving maintenance rather than a static prompt drop. The README is candid about its own benchmarks, revising an earlier headline figure downward after issue feedback, which suggests the numbers are maintained rather than frozen marketing.

## Alternatives

- **[caveman](https://github.com/JuliusBrussee/caveman)** — use it when the problem is how much the agent *says*, not how much it *builds*; the README frames the two as complementary and non-overlapping.
- A hand-maintained "YAGNI + one-liners" prompt or project ruleset — use it when you want full control over the wording and are willing to give up the multi-agent adapters, review commands, and safety carve-outs; the README benchmarks this as a separate, lower-scoring arm.

## Notes

- The published cost, token, and speed reductions come from a benchmark against a fair agentic baseline (a headless Claude Code session editing a real FastAPI + React repo), and the README explicitly retracts an earlier 80–94% single-shot figure as partly a conversational-baseline artifact.
- Uninstalling via the host leaves behind state (mode flag, config file, an optional statusline entry); a separate `scripts/uninstall.js` is provided and is meant to be run before the host's own remove command, since removing the plugin first deletes the script.

## Tags

javascript, cli, developer-tools, ai-agents, claude-code, prompt-engineering, llm, agent-skills, code-generation, plugin
