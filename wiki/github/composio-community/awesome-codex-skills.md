# composio-community/awesome-codex-skills

A curated list of practical Codex skills, with a Python installer that drops them into `$CODEX_HOME/skills`.

## What it is

This repository is an awesome-list of modular instruction bundles ("skills") for OpenAI's Codex CLI and API, aimed at developers who want to extend Codex with reusable, task-specific behaviors. Each skill is a folder containing a `SKILL.md` with name/description frontmatter and step-by-step guidance; Codex reads the metadata to decide when to trigger a skill and loads the body only after it fires, keeping context lean. Beyond the curated index, the repo bundles many skill folders directly and a helper script to install them from GitHub.

## Key features

- A categorized index of skills across Development & Code Tools, Productivity & Collaboration, Communication & Writing, Data & Analysis, and Meta & Utilities.
- A Python skill installer (`skill-installer/scripts/install-skill-from-github.py`) that fetches a skill by repo and path into `$CODEX_HOME/skills` (default `~/.codex/skills`).
- In-repo skill folders (each with its own `SKILL.md`), plus a large `composio-skills/` directory of per-app automation skills.
- A `template-skill/` starter and a `skill-creator/` guide describing the layout (SKILL.md plus optional scripts/, references/, assets/) and progressive-disclosure practices.
- Composio-oriented "connect" skills for wiring Codex to external apps (Slack, GitHub, Notion, and others) via the Composio CLI.

## Tech stack

- Primary language Python, used for the skill-installer scripts; skills themselves are Markdown (`SKILL.md`) with YAML frontmatter.
- Targets the Codex CLI and API; skills install under `$CODEX_HOME/skills` and are triggered by their `description` metadata after a Codex restart.
- Some listed skills provide their own scripts or MCP servers (for example a local PPTX/DOCX/XLSX generator and various MCP-based skills); these are external and vary by skill.
- No root build manifest and no declared license file (SPDX: none).

## When to reach for it

- Finding and installing ready-made Codex skills instead of writing task instructions from scratch.
- Learning the SKILL.md format and best practices before authoring your own skill (via `template-skill/` and `skill-creator/`).
- Connecting Codex to external SaaS apps for real actions through Composio's CLI.
- Browsing per-app automation skills in one place.

## When *not* to reach for it

- It is a curated list and skill collection, not a runtime or framework — it does nothing on its own beyond installing Markdown instruction bundles.
- Skills are Codex-specific; the format and installer target Codex's `$CODEX_HOME/skills`, so it is not a general cross-agent registry (though some listed skills separately note Claude Code or other support).
- The repository has no declared license, which can be a blocker for organizations that require explicit licensing before reuse.
- Many entries link to third-party external repos of varying quality and maintenance; inclusion in the list is not a guarantee of vetting.

## Maturity signal

Created in January 2026, the list drew a large audience quickly, but its last push (May 2026) is several months before this writing, so it is more a curated snapshot than a daily-updated project — a common pattern for awesome-lists once a critical mass of entries lands. The open-issue count is moderate and the default branch is still `master`. The absence of a license file is the main caution: high visibility, but reuse terms are unstated and updates appear to have slowed.

## Alternatives

- awesome-claude-code — use it when your agent is Claude Code rather than Codex and you want a comparable community skill/resource index.
- The Composio platform and CLI — use it directly when you mainly want app connectors and real actions rather than a skills index.
- The official Codex documentation — use it when you want authoritative guidance on the skills format rather than a community collection.

## Notes

- Codex loads only a skill's metadata up front and reads the full body only when the skill triggers, which is why descriptions are meant to be exhaustive about *when* to trigger while bodies stay focused on execution steps.
- The repo is much larger than the README index suggests: a `composio-skills/` directory holds a very large number of per-app "-automation" skill folders (well over a thousand paths), far beyond the human-curated lists in the README.

## Tags

awesome-list, codex, llm, agent-skills, python, automation, mcp, developer-tools
