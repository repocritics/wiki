# JCodesMore/ai-website-cloner-template

A reusable template that drives an AI coding agent to reverse-engineer a live website into a clean Next.js codebase.

## What it is

This is a GitHub template repository plus a `/clone-website` skill that turns a URL into a modern Next.js project by having an AI coding agent inspect the site, extract design tokens and assets, write component specs, and reconstruct each section. It is intended for developers who want to rebuild a site they own — for migration off a legacy or no-code platform, to recover lost source, or to learn how a production site is built — by working from real, generated code rather than guessing. Claude Code with Opus 4.8 is recommended, but a range of agents are supported.

## Key features

- A `/clone-website` skill that runs a multi-phase pipeline: reconnaissance (screenshots, design-token extraction, interaction sweep), foundation (fonts, colors, globals, asset download), component specs, parallel build, and assembly with visual QA.
- Parallel build dispatches builder agents in separate git worktrees, one per section or component, each given an inline specification with exact computed CSS values, states, behaviors, and asset paths.
- Broad agent support (Claude Code recommended; Codex CLI, OpenCode, Copilot, Cursor, Windsurf, Gemini CLI, Cline, Roo Code, Continue, Amazon Q, Augment Code, and Aider listed as supported).
- Single-source-of-truth instruction files (`AGENTS.md` and the `clone-website` `SKILL.md`) that sync out to each platform's config via `scripts/sync-agent-rules.sh` and `scripts/sync-skills.mjs`.
- A conventional Next.js project scaffold (routes, components, shadcn/ui primitives, extracted icons, downloaded images/videos, SEO assets) ready for the generated output.

## Tech stack

- TypeScript on Next.js 16 (App Router) with React 19.2.4 and strict TypeScript; `next` 16.2.1 and `eslint-config-next` 16.2.1.
- shadcn/ui (`shadcn` ^4.1.0) with `@base-ui/react` ^1.3.0 and Tailwind CSS v4 (`@tailwindcss/postcss` ^4) using oklch design tokens.
- Utility deps: `class-variance-authority`, `clsx`, `tailwind-merge` ^3.5.0, `tw-animate-css` ^1.4.0, and `lucide-react` ^1.6.0 for default icons.
- Node.js 24+ required (`engines.node >=24`); MIT-licensed, package version 0.3.1. Optional Docker and Docker Compose setup (`app` and `dev` services).

## When to reach for it

- Rebuilding a site you own from WordPress, Webflow, or Squarespace into a maintainable Next.js codebase.
- Recovering a modern codebase for a live site whose source is lost or on a legacy stack.
- Studying how a production site achieves specific layouts, animations, and responsive behavior by working with real code.

## When *not* to reach for it

- You need a static offline mirror of a site rather than a rebuilt, componentized codebase.
- The project explicitly excludes phishing, impersonation, passing off someone else's design as your own, and scraping sites whose terms of service prohibit it.
- You want a finished product with no agent in the loop — this is a template that orchestrates an AI coding agent, not a one-click converter, and output still needs review and customization.

## Maturity signal

The template attracted a very large audience within months of its early-2026 creation, and was pushed to within weeks of this writing, so it is active and clearly resonating. Its low open-issue count and early version (0.3.1) point to a young, still-evolving project rather than a settled tool; the MIT license and template structure make it easy to fork and adapt. Treat it as an actively developed, fast-moving template whose conventions may still shift.

## Alternatives

- HTTrack (or `wget`) — use instead when you just want to mirror a site's existing HTML/CSS/assets offline, not regenerate it as a modern component codebase.
- Firecrawl — use instead when you need to scrape a site's content and structure as data for an LLM rather than reconstruct its UI.
- v0 — use instead when you want to generate UI from a prompt or a screenshot rather than faithfully clone a specific live site.

## Notes

- It is meant to be consumed via GitHub's "Use this template" button, not cloned directly; the README warns against opening pull requests with generated websites back to the template.
- The design-fidelity approach leans on `getComputedStyle()` values captured during reconnaissance and passed inline to each builder agent, so builders are told not to guess.
- Two sync scripts regenerate all per-platform instruction and skill files from two source-of-truth files, which is how the same skill supports over a dozen different agents.

## Tags

typescript, nextjs, react, tailwindcss, template, ai-agents, web-scraping, reverse-engineering, boilerplate, claude-code
