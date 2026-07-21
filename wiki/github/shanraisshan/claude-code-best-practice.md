# shanraisshan/claude-code-best-practice

A curated reference and reference-implementation repository documenting how to configure and operate Claude Code, spanning subagents, slash commands, skills, hooks, MCP, settings, and multi-step development workflows.

## What it is

This is a documentation and example repository, not an application. It catalogs Claude Code's configuration surfaces and features, links each one to the official Anthropic docs, and pairs many of them with a "best practice" writeup and a worked "implemented" example living under `.claude/`. The audience is developers adopting Claude Code who want a single map of what the tool can do and concrete file layouts to copy from. It also surveys the wider ecosystem, listing third-party workflow methodologies, cross-model bridges, skill libraries, and agent libraries with star counts so readers can compare options.

## Key features

- A "Concepts" catalog that maps each Claude Code primitive (subagents at `.claude/agents/<name>.md`, commands at `.claude/commands/<name>.md`, skills at `.claude/skills/<name>/SKILL.md`, hooks, MCP servers, settings, memory, checkpointing, CLI flags) to its docs and to local best-practice and implementation files.
- A worked orchestration example demonstrating the Command → Agent → Skill pattern, runnable via `claude` then `/weather-orchestrator`, with an accompanying diagram and demo.
- A "Development Workflows" comparison table of external methodologies (Superpowers, Everything Claude Code, Spec Kit, BMAD-METHOD, oh-my-claudecode, and others), each framed against a Research → Plan → Execute → Review → Ship pattern with agent/command/skill counts.
- A "Cross-Model Workflows" section explaining three ways to combine Claude Code with other models (plugin, MCP, router) and listing concrete projects for each.
- Curated "Skill Collections" and "Agent Collections" tables ranking third-party `SKILL.md` and subagent libraries by star count.
- A working `.claude/` directory with example agents, commands, and a Python hooks script (`.claude/hooks/scripts/hooks.py`) plus per-event sound assets.

## Tech stack

- Primary language reported as HTML (rendered thumbnail and presentation pages under `!/thumbnail/` and `!/video-presentation-transcript/`).
- Markdown for all reference and best-practice documents.
- Python for the hooks implementation (`.claude/hooks/scripts/hooks.py`) driven by `.claude/hooks/config/hooks-config.json`.
- Claude Code configuration files: `.claude/agents/`, `.claude/commands/`, `.claude/skills/`, `.claude/settings.json`, and `.mcp.json`.
- No package manifests are present at the repository root, so there is no build or dependency toolchain to install.

## When to reach for it

- You are setting up Claude Code and want a single index of its features linked to the official docs.
- You want copy-ready examples of the file layout for subagents, commands, skills, hooks, and MCP configuration.
- You want to study the Command → Agent → Skill orchestration pattern against a concrete, runnable example.
- You are comparing third-party Claude Code workflow methodologies, skill libraries, or agent libraries and want them collected in one place with relative popularity.

## When *not* to reach for it

- You want a library or framework to import into your own code; this repository ships reference material and examples, not a published package.
- You want an authoritative API contract; it links out to Anthropic's official documentation, which is the source of truth for exact behavior.
- You are not using Claude Code; the material is specific to that tool's configuration model and has little value for other agents or editors.
- You need vendor-neutral guidance; the content is centered on Claude Code and Anthropic's ecosystem.

## Maturity signal

The repository is very actively maintained: it was created in late October 2025 and, per the repo facts, was last pushed the same day this page reflects, with the README carrying a dated "updated with Claude Code" badge. Roughly 63,000 stars and a "GitHub Trending #1 of the day" claim indicate it caught significant attention quickly rather than accreting over years, which is consistent with its creation date. With only a handful of open issues and an MIT license, it reads as an energetically curated reference that tracks Claude Code's fast-moving feature set. The main risk is not abandonment but staleness relative to upstream, since much of its value is links and examples that must be updated as Claude Code changes.

## Alternatives

- Use the official Claude Code documentation at code.claude.com/docs instead when you need the authoritative, always-current specification of any feature rather than a curated index.
- Use anthropics/skills instead when you specifically want Anthropic's own maintained set of `SKILL.md` files to install directly.
- Use obra/superpowers instead when you want an opinionated end-to-end workflow methodology (brainstorming through code review) rather than a broad catalog.
- Use github/spec-kit instead when you want a spec-driven command workflow (`/speckit.*`) to structure a project rather than reference material.

## Notes

- Despite the topic being agentic coding, the repository's primary language is reported as HTML, reflecting its nature as a documentation and presentation artifact rather than a codebase.
- The star counts shown in the ecosystem tables are unusually large (for example a workflow listed at 258k), suggesting the tables use a scaled or aggregated figure rather than raw GitHub stars; treat them as relative rankings.
- Several linked "implementation" and "best-practice" targets point to separate repositories under the same owner (hooks, status line, self-evolving loop), so the project is one node in a set of related repos.

## Tags

claude-code, awesome-list, documentation, agentic-coding, llm, ai-agents, best-practices, anthropic, prompt-engineering, mcp, developer-tools
