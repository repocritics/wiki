# ComposioHQ/awesome-claude-skills

A curated index of over a thousand Claude Skills and plugins, plus pre-built app-automation skills, for use across Claude and other coding agents.

## What it is

This is an awesome-list repository that catalogs reusable Claude Skills, the SKILL.md instruction packages that teach an agent how to handle a class of tasks. It exists to help users of Claude Code, Claude.ai, the Claude API, and compatible agents discover an existing skill before writing their own. Entries are grouped into categories such as Document Processing, Development & Code Tools, Data & Analysis, Business & Marketing, and Security & Systems, and each links out to the skill's source with author attribution. Alongside the curated links, the repo ships its own connect-apps plugin and a large set of per-app automation skills.

## Key features

- Catalogs a stated 1000+ skills and plugins organized into task categories, with each entry linking to its external repository and crediting the author.
- Documents the SKILL.md format and its progressive-loading model: name and description (roughly 100 tokens) load at session start, the full body (typically under 5,000 tokens) loads when the agent judges the skill relevant, and files under `scripts/` and `references/` load on demand.
- Ships a connect-apps plugin that lets Claude take real actions across a stated 500+ apps via Composio, installed with `claude --plugin-dir ./connect-apps-plugin` and configured through `/connect-apps:setup` with a Composio API key.
- Includes an "App Automation via Composio" section of pre-built workflow skills for 78 SaaS apps through Rube MCP, each with tool sequences, parameter guidance, known pitfalls, and quick-reference tables using real Composio tool slugs.
- Targets not only Claude but other agents the README names as supporting the format: OpenAI Codex, Cursor, Gemini CLI, Antigravity, and Windsurf.
- Contains a large `composio-skills/` tree of individual `*-automation/SKILL.md` folders plus a Claude plugin marketplace manifest at `composio-skills/.claude-plugin/marketplace.json`.

## Tech stack

- Built around Markdown as a content repository (the README is the primary index); no recognized root manifest such as `package.json` or `pyproject.toml` was found.
- SKILL.md format: YAML frontmatter (name, description) plus Markdown instructions, optionally bundled with `scripts/`, `references/`, and assets.
- Primary language reported as Python; some bundled skills carry shell scripts (for example `artifacts-builder/scripts/*.sh`) and binary assets such as fonts and tarballs.
- connect-apps plugin for Claude Code, configured via `/connect-apps:setup` and a Composio API key.
- App-automation skills rely on Rube MCP (Composio) tool slugs.
- Claude plugin marketplace manifest at `composio-skills/.claude-plugin/marketplace.json`.
- The README license badge declares Apache-2.0; the repository SPDX metadata is unset.

## When to reach for it

- Looking for an existing skill for a task before building one from scratch.
- Wiring up real app actions (email, issue creation, Slack messages, CRM updates) for Claude through the connect-apps plugin.
- Learning the SKILL.md format and how skills load progressively into an agent's context.
- Browsing per-app automation recipes for one of the 78 supported SaaS tools.

## When *not* to reach for it

- When you want the canonical, first-party spec and skills, go to the upstream `anthropics/skills` repository the format comes from rather than a third-party index.
- When you need a runtime, framework, or library to import; this is an index and content repository, not executable infrastructure.
- When your agent does not support the SKILL.md format, older or non-compatible tools cannot consume these entries.
- When you need vendor-neutral app automation, the App Automation half is tied to Rube MCP and Composio.

## Maturity signal

Created in October 2025 and last pushed in May 2026, the repo reached roughly 68,000 stars and 7,700 forks in well under a year, which points to unusually fast adoption and active curation rather than a dormant list. The 1,000-plus open issues most likely reflect the submission and pull-request volume typical of a high-traffic awesome list rather than defects. It is not archived and is backed by a company (Composio). The license is nominally Apache-2.0 per the README badge, though the repository's SPDX metadata is unset.

## Alternatives

- `anthropics/skills` — the upstream open standard and first-party skills; use it when you want canonical, Anthropic-maintained skills instead of a curated third-party catalog.
- `obra/superpowers` — an opinionated engineering-workflow skill collection (test-driven development, git worktrees, brainstorming, root-cause tracing); use it when you want a focused developer pack rather than a broad directory.
- `mhattingpete/claude-skills-marketplace` — a marketplace-structured multi-plugin skills repo; use it when you want grouped, installable plugins over a flat list of links.

## Notes

- Despite the Claude framing, the list explicitly serves other agents: the README names Codex, Cursor, Gemini CLI, Antigravity, and Windsurf as supporting the same format.
- The README draws a firm distinction between the layers: skills define workflow and behavior, MCP defines access, and tools are the invocable functions; skills are not MCP servers and not tools, and in production all three run together.
- Much of the repo's file volume lives in the `composio-skills/` tree of hundreds of `*-automation/SKILL.md` folders, which is larger than the hand-curated README list itself.
- The README states Anthropic introduced the SKILL.md format in October 2025 and released it as an open standard in December 2025.

## Tags

awesome-list, claude, claude-code, ai-agents, agent-skills, mcp, automation, developer-tools, saas, workflow-automation, llm, python
