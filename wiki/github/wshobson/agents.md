# wshobson/agents

A single-source marketplace of agents, skills, and commands for Claude Code that also generates native artifacts for Codex CLI, Cursor, OpenCode, Gemini CLI, and GitHub Copilot.

## What it is

wshobson/agents is an agentic plugin marketplace that packages reusable workflow building blocks — agents, skills, commands, and orchestrators — as Markdown from one source of truth in `plugins/`. From that source it emits harness-native artifacts for five additional harnesses rather than lowest-common-denominator translations, so the same content works idiomatically across Claude Code, OpenAI Codex CLI, Cursor, OpenCode, Gemini CLI, and GitHub Copilot. It is aimed at developers who want production-ready domain experts and slash commands they can install per-plugin instead of adopting a whole monolith.

## Key features

- A catalog of 94 plugins, 203 agents, 175 skills, 109 commands, and 16 multi-agent orchestrators, per the README's own counts.
- Granular, single-purpose plugins; installing a plugin loads only its components into context, not the whole marketplace.
- Multi-harness generation from one Markdown source, with per-harness adapters (for example `.codex/`, `.cursor/rules/`, `.opencode/`, Gemini TOML, `.copilot/`) driven by `make generate-all`.
- A tiered model strategy that maps each agent to a model tier (Fable 5, Opus, inherit, Sonnet, Haiku) by task type.
- `plugin-eval`, a three-layer quality framework: deterministic static analysis, an LLM-judge across four dimensions, and Monte Carlo reliability runs.
- Make-based tooling for structural validation and drift/dead-link/cap detection (`make validate`, `make garden`).

## Tech stack

- Primary language reported as Python, used by the `plugin-eval` framework and run via `uv` (`uv run plugin-eval ...`).
- Content authored as Markdown (agents, skills as `SKILL.md`, commands) as the source of truth.
- JSON marketplace/plugin registries per harness (for example `.claude-plugin/marketplace.json`, `.agents/plugins/marketplace.json`, `.cursor-plugin/`), and TOML output for Gemini CLI.
- A `Makefile` orchestrates generation, validation, and installation targets.
- MIT licensed.

## When to reach for it

- You use Claude Code and want a large library of ready-made domain agents, skills, and slash commands.
- You work across more than one agentic harness and want the same content available natively in each.
- You want to install narrowly scoped plugins so only relevant components enter the model's context.
- You want a repeatable way to measure and certify the quality of agents and skills before shipping them.

## When *not* to reach for it

- You need runtime multi-agent orchestration logic; this is primarily a content marketplace, not an execution engine.
- You want a plug-and-play binary; Gemini and OpenCode install via clone plus `make generate`, and some trees are gitignored and must be built.
- You only ever use one harness and prefer that harness's own first-party plugin ecosystem.
- You need harness features beyond the documented capability matrix; some deltas (for example an 8 KB skill cap on Codex) are respected by trimming or remapping content.

## Maturity signal

The project has over 38,000 stars gathered since its creation in July 2025, a most-recent push in July 2026, and only 7 open issues, which together read as very actively maintained with a well-tended issue tracker. The MIT license and single-source, multi-harness architecture suggest a project built for ongoing extension rather than a one-off dump, and the presence of a dedicated evaluation framework and CI workflows (validate, code-quality, eval-report) reinforces active quality control.

## Alternatives

- Yeachan-Heo/oh-my-claudecode — use it instead when you want a runtime multi-agent orchestration system rather than a catalog of installable content.
- Claude Code's native subagents and plugins — use them instead when you want to stay entirely within first-party tooling and define agents yourself.
- Community "awesome" Claude Code lists — use them instead when you want a curated index of resources rather than installable, harness-native plugins.

## Notes

The five non-Claude harnesses receive purpose-built artifacts, not shared translations: Codex and Cursor install from committed registries that point back at the source `plugins/`, while Gemini and OpenCode install through clone-plus-generate with their transformed trees gitignored. The repo also includes an external memory integration, Pensyve, wired in as a `git-subdir` entry with separate upstream integrations for each supported harness.

## Tags

python, markdown, claude-code, ai-agents, prompt-engineering, developer-tools, orchestration, mcp, multi-agent, cli
