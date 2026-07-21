# kepano/obsidian-skills

A set of agent skills that teach AI coding agents to work with Obsidian's CLI and open file formats.

## What it is

obsidian-skills is a collection of Agent Skills that let any skills-compatible AI agent create and edit Obsidian content and drive an Obsidian vault. It packages instructions for Obsidian Flavored Markdown, Bases, JSON Canvas, the Obsidian CLI, and web-content extraction, so agents produce valid Obsidian-specific syntax instead of guessing. It is aimed at Obsidian users who run agents such as Claude Code, Codex, or Open Code against their vaults.

## Key features

- `obsidian-markdown` — create and edit Obsidian Flavored Markdown with wikilinks, embeds, callouts, and properties.
- `obsidian-bases` — author Obsidian Bases (`.base`) with views, filters, formulas, and summaries.
- `json-canvas` — create and edit JSON Canvas (`.canvas`) files with nodes, edges, groups, and connections.
- `obsidian-cli` — interact with vaults through the Obsidian CLI, including plugin and theme development.
- `defuddle` — extract clean Markdown from web pages to reduce token use.
- Built to the Agent Skills specification, so the same skills work across multiple compatible agents.

## Tech stack

- Content is authored as `SKILL.md` files following the Agent Skills specification (no application source language).
- Distributed as a Claude Code plugin (`.claude-plugin/marketplace.json`, `plugin.json`) and via `npx skills`.
- Reference material for callouts, embeds, properties, functions, and examples ships alongside each skill.
- Targets the Obsidian CLI and Obsidian open formats (Markdown, Bases, JSON Canvas); the `defuddle` skill uses the Defuddle extractor.

## When to reach for it

- You use Obsidian and want an agent to reliably produce correct Obsidian syntax and file formats.
- You want to automate vault edits, Bases, or Canvas files through a coding agent.
- You are on a skills-compatible agent (Claude Code, Codex, Open Code) and want a drop-in install.
- You want agents to save web content into your vault as clean Markdown.

## When *not* to reach for it

- You do not use Obsidian or its open formats — the skills are Obsidian-specific.
- Your agent does not support the Agent Skills specification.
- You need a runtime plugin or automation service rather than instructions an agent reads.
- You want general note-taking automation independent of Obsidian's syntax and CLI.

## Maturity signal

Created in early January 2026 and last pushed in June 2026, the repository has a large following and a small, focused surface of five skills. The roughly six-week gap between the last push and this writing, combined with a low open-issue count and a stable specification-based format, suggests a compact project in light maintenance rather than one under constant churn. MIT licensing and authorship by a prominent Obsidian community figure lend it credibility.

## Alternatives

- sickn33/agentic-awesome-skills — use it when you want a broad, general-purpose catalog of agent skills rather than an Obsidian-focused set.
- Obsidian MCP servers — use one when you want a running server exposing vault operations to agents instead of skill instructions.
- Obsidian community plugins — use these when you want in-app automation without an external AI agent.

## Notes

The README warns that Open Code users must clone the whole repository (not just the inner `skills/` folder) so the directory structure resolves correctly, and notes that Open Code auto-discovers `SKILL.md` files with no config changes. The repository carries no application manifests — it is purely skill content and packaging.

## Tags

agent-skills, obsidian, markdown, cli, claude, codex, knowledge-management, json-canvas, developer-tools, llm
