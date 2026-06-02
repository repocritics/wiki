# obra/superpowers

An "agentic skills framework + software-development methodology" aimed at coding agents like Claude Code, Codex, Cursor, and other CLI agents.

## What it is

A set of composable shell-driven skills plus an initial-instruction layer that, per the README, gets coding agents to follow a consistent methodology rather than freestyling each session. The pitch is that an agent equipped with these "superpowers" works through software-development tasks in a more structured way — recipe-style skills are invoked from the agent's main loop, with the framework owning context-management and skill dispatch. Distributed as a project to install into a target agent's runtime.

## Key features

- Integrations called out in the README: Claude Code, Codex CLI, Codex App, Factory Droid, Gemini CLI, OpenCode, Cursor, GitHub Copilot CLI.
- Composable skill model — discrete shell-based skills the agent can compose into workflows.
- "Initial instructions" layer that pins the agent to a defined methodology rather than ad-hoc reasoning.
- Shell-first implementation, low dependencies.
- MIT-licensed.

## Tech stack

- Shell scripts are the primary implementation; skills are shell-callable units.
- Designed for compatibility with multiple agent harnesses rather than coupling to one runtime.

## When to reach for it

- You're driving an agent (Claude Code, Codex, Cursor) day-to-day and find session-to-session inconsistency a tax.
- You want a methodology you can point at when reviewing what an agent did, not just outputs.
- You're researching agent-skill composition patterns.

## When *not* to reach for it

- You want a fully-managed agent product — this is a layer to install on top of an existing agent, not a standalone agent.
- You don't run shell on your dev machine — the skill substrate is shell-based.
- You're risk-averse about installing third-party shell scripts that drive your coding agent. Audit the installed surface before integrating.

## Maturity signal

215k stars accumulated since October 2025 puts this in fast-rising territory. Last push the day of generation (2026-06-02) — active. MIT license clean. Anyone evaluating this should weigh the velocity carefully: rapid star growth in the agent-skills niche has been a hot space and a target for hype cycles, so reading the actual skill source code before installing it on a credentialed agent is warranted.

## Alternatives

- `anthropics/skills` and similar first-party skill bundles — use when you want vendor-provided baseline rather than a third-party layer.
- LangChain / LlamaIndex agent abstractions — use when you want library-style agent composition with Python.
- Plain CLAUDE.md / AGENTS.md files in your project — use when you want zero-framework, just per-project conventions.

## Notes

The agent-skills category emerged rapidly in late 2025 and saw a wave of high-star-velocity entrants. Treat star count alone as low signal in this niche; review the installed shell surface, the skill catalog, and the upstream commit history before plugging this into an agent that has filesystem or credential access. License absence isn't an issue here (MIT) but supply-chain trust still requires direct code review.

## Tags

artificial-intelligence, large-language-model, agent, claude, cursor, command-line-interface, framework, developer-tools, shell
