# coreyhaines31/marketingskills

A collection of Agent Skills that give AI coding agents specialized workflows for marketing tasks such as CRO, copywriting, SEO, analytics, and growth engineering.

## What it is

Marketing Skills is a set of markdown-based skills that teach AI agents how to handle specific marketing jobs. Each skill is a `SKILL.md` file (often with reference material and evals) that an agent reads when it recognizes a matching task, so it applies the right framework instead of improvising. It is built for technical marketers and founders who use agents like Claude Code, Codex, Cursor, or Windsurf and want them to help with conversion, content, SEO, and measurement work.

## Key features

- Roughly four dozen skills spanning conversion optimization, content and copy, SEO and discovery, paid and distribution, measurement and testing, retention, growth engineering, strategy, and sales/RevOps.
- A shared `product-marketing` skill that every other skill reads first to ground work in your product, audience, and positioning.
- Skills that cross-reference each other (for example copywriting ↔ cro ↔ ab-testing), with a Related Skills section in each documenting the dependency map.
- Multiple install paths: `npx skills`, a Claude Code plugin/marketplace, clone-and-copy, git submodule, fork, or `npx skillkit` for multi-agent installs.
- Direct invocation of skills (for example `/cro`, `/emails`, `/seo-audit`) in addition to automatic task recognition.

## Tech stack

- Content is authored as markdown `SKILL.md` files, each with optional `references/` and `evals/evals.json`.
- Primary language reported as JavaScript, used for repository automation such as `.github/scripts/sync-skills.js` and skill-validation workflows.
- Packaged as a Claude Code plugin via `.claude-plugin/marketplace.json` and `plugin.json`.
- Targets the Agent Skills spec (agentskills.io) and installs into `.claude/skills/` or a shared `.agents/skills/` directory.
- License: MIT. No root dependency manifest was included in the inputs.

## When to reach for it

- Giving an AI coding agent repeatable, framework-backed guidance for marketing tasks rather than ad-hoc prompts.
- Working across several marketing disciplines (SEO, copy, ads, analytics, retention) from a single installed skill set.
- Standardizing how a small team or founder runs conversion, content, and growth work through agents.

## When *not* to reach for it

- You are not using an agent that reads the Agent Skills spec; these are instruction bundles, not a runnable application or library.
- Your marketing workflows are highly company-specific and would need heavy customization beyond the general frameworks here.
- You want fully autonomous execution of campaigns; the skills guide an agent's reasoning but still assume human review and judgment.

## Maturity signal

The project is recent but has drawn a very large star count quickly and shows a recent push date, pointing to an active, popular repository rather than a stalled one. The low open-issue count relative to its popularity suggests the content-focused scope keeps maintenance manageable. The MIT license is permissive. A documented v1.x-to-v2.0 migration that renamed 17 skills and merged others indicates the collection is being actively restructured, so pinning to a version matters when upgrading.

## Alternatives

- Custom, project-specific skills or `CLAUDE.md` instructions — reach for these when your marketing processes are proprietary enough that generic frameworks do not fit.
- General-purpose prompt collections (for example `f/awesome-chatgpt-prompts`) — reach for these when you want raw copy-paste prompts rather than agent-triggered skills tied to task recognition.

## Notes

- Installing from inside an agent session can be a trap: the CLI may run non-interactively and only write to the universal `.agents/skills/` directory, which Claude Code does not read, so the README advises passing the agent explicitly (`-a claude-code`).
- The README is candid that upgrading to v2.0 leaves stale old-name folders behind and gives an explicit cleanup command.

## Tags

marketing, ai-agents, claude-code, skills, seo, copywriting, conversion-optimization, growth, prompt-engineering, markdown, awesome-list
