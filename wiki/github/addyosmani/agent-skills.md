# addyosmani/agent-skills

A pack of engineering "skills" that give AI coding agents structured, verification-gated workflows across the full software lifecycle.

## What it is

Agent Skills is a collection of Markdown-defined workflows that encode the steps, quality gates, and practices senior engineers use, packaged so AI coding agents follow them consistently. It addresses the tendency of coding agents to take the shortest path and skip specs, tests, and reviews. It is aimed at developers and teams who want their agents to apply the same discipline across defining, planning, building, verifying, reviewing, and shipping code.

## Key features

- Eight slash commands mapped to the development lifecycle (`/spec`, `/plan`, `/build`, `/test`, `/review`, `/webperf`, `/code-simplify`, `/ship`), with a `/build auto` mode that plans and implements every task in one approved, still test-driven pass.
- 24 skills total (23 lifecycle skills plus a `using-agent-skills` meta-skill) that also activate automatically based on the task at hand.
- A consistent skill anatomy: overview, when-to-use, process steps, anti-rationalization tables of common excuses, red flags, and mandatory verification evidence.
- Four pre-configured specialist agent personas (code reviewer, test engineer, security auditor, web performance auditor) and seven reference checklists.
- Installs across many tools via the `skills` CLI or native plugins for Claude Code, Cursor, Codex, Copilot, Gemini CLI, Antigravity, Windsurf, OpenCode, Kiro, and others.

## Tech stack

- Skills, agents, and checklists authored as plain Markdown, so they work with any agent that accepts instruction files.
- JavaScript tooling for validation and evaluation (skill linting, command/skill validators, an eval runner over case fixtures).
- Distribution via the `npx skills` CLI and plugin manifests for multiple ecosystems (`.claude-plugin`, `.codex-plugin`, Gemini `.gemini/commands` TOML, an Antigravity `plugin.json`, and per-tool docs).
- Session lifecycle hooks under `hooks/`. No pinned dependency manifest is present in the repository root, so specific version constraints are not declared here.

## When to reach for it

- You want to enforce consistent engineering discipline (specs before code, tests as proof, reviews before merge) on AI agents.
- You work across several agents or IDEs and want one shared, portable set of workflows.
- You want lifecycle coverage from idea refinement through shipping, with explicit verification gates.

## When *not* to reach for it

- You prefer minimal, ad hoc prompting and find opinionated multi-step process overhead for small changes.
- Your work is outside software engineering; the skills encode software delivery practices specifically.
- You want a runtime framework or library rather than a set of instruction files for agents to follow.

## Maturity signal

The project has accumulated a very large star count within a few months of creation and was pushed close to the snapshot date, indicating active maintenance and fast adoption. It is MIT-licensed, not archived, and maintained by a named author and collaborators, with a comparatively modest open-issue count for its size. The overall picture is an actively maintained project rather than one in maintenance-only mode.

## Alternatives

- obra/superpowers — use it instead when you want a differently shaped skills pack; the repository's own comparison doc contrasts the two directly.
- mattpocock/skills — use it instead when you prefer that maintainer's take on agent skills, also covered in the comparison doc.

## Notes

The skills deliberately bake in ideas from Google's engineering practices — Hyrum's Law in API design, the Beyonce Rule and test pyramid in testing, Chesterton's Fence in simplification, trunk-based development, and Shift Left — embedded into step-by-step workflows rather than presented as abstract advice. The repository also ships an `evals/` suite with per-skill cases and fixtures to test the skills themselves.

## Tags

ai-agents, agent-skills, claude-code, developer-tools, workflow, code-review, test-driven-development, markdown, javascript
