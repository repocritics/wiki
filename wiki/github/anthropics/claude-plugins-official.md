# anthropics/claude-plugins-official

Anthropic's curated marketplace repository of Claude Code plugins, split into internally maintained and third-party entries.

## What it is

This repository is the official directory that Claude Code reads to discover and install plugins — bundles of slash commands, agents, skills, and MCP server configurations. It serves Claude Code users who want vetted plugins and plugin authors who want their work listed. Anthropic maintains the internal plugins under `/plugins` and reviews third-party submissions under `/external_plugins`, while noting it does not control what third-party MCP servers or code the external plugins pull in.

## Key features

- A marketplace catalog (`.claude-plugin/marketplace.json`) that Claude Code installs from via `/plugin install {name}@claude-plugins-official`.
- A standard plugin layout: `.claude-plugin/plugin.json` metadata, an optional `.mcp.json`, and `commands/`, `agents/`, and `skills/` directories.
- Immutable plugin `name` slugs, with a top-level `renames` map so a renamed plugin auto-migrates existing installs.
- Skill-bundle entries that expose `SKILL.md` directories directly using `strict: false` and an explicit `skills` array pointing at subdirectories of a source repo.
- A split between Anthropic-maintained internal plugins and reviewed external/community plugins.
- CI workflows for validating frontmatter, licenses, plugins, and MCP URLs, and for bumping plugin commit SHAs.

## Tech stack

- Primarily a data and configuration repository: plugin and marketplace metadata in JSON, skills and agents in Markdown.
- Repository tooling in Python (for example `discover_bumps.py`), TypeScript (`validate-frontmatter.ts`), and JavaScript, run through GitHub Actions workflows.
- Individual external plugins ship their own stacks — several include `package.json`, `bun.lock`, and a `server.ts`, indicating Bun/TypeScript MCP servers.
- No root build manifest; the repository is consumed by Claude Code's plugin loader rather than built.

## When to reach for it

- Installing vetted plugins into Claude Code from a trusted, Anthropic-managed source.
- Publishing a plugin or skill bundle you want discoverable through Claude Code's `/plugin > Discover` flow.
- Learning the expected plugin structure by reading the reference `example-plugin` and existing entries.
- Wiring MCP servers, slash commands, agents, or skills into Claude Code as reusable packages.

## When *not* to reach for it

- Any tool other than Claude Code — the marketplace format and loader are Claude Code specific.
- Situations requiring guaranteed safety of third-party plugins; the README states Anthropic cannot verify external plugins or their MCP servers.
- Distributing private or internal-only plugins, which belong in a self-hosted marketplace rather than this public directory.

## Maturity signal

Despite being created in late 2025, the repository has passed 32,000 stars and is pushed to within the last day, reflecting the rapid adoption of Claude Code's plugin system rather than a mature codebase. The high open-issue count and heavy CI automation around validation and SHA bumping suggest an actively curated, fast-moving catalog. It is Apache-2.0 licensed, though each linked plugin carries its own license.

## Alternatives

- modelcontextprotocol/servers — use instead when you want a reference collection of MCP servers rather than Claude Code plugin packages.
- Smithery — use instead when you want a hosted registry to discover and install MCP servers across multiple clients.
- A self-hosted marketplace using the same `marketplace.json` schema — use instead when you need to distribute private or organization-internal plugins.

## Notes

Plugin `name` fields are treated as permanent identifiers: renaming one breaks existing installs unless an entry is added to the `renames` map, which the loader uses to transparently migrate users on their next sync. The `strict: false` skill-bundle mechanism also lets the marketplace expose skills from repositories that never shipped a `plugin.json`.

## Tags

claude-code, plugins, marketplace, mcp, skills, developer-tools, json, directory, automation
