# thedaviddias/Front-End-Checklist

An open-source front-end quality checklist of hundreds of rules, usable in the browser, in the README, or through an MCP server for AI agents.

## What it is

Front-End Checklist turns front-end best practices into a practical review workflow for both humans and AI agents. It provides a corpus of rules across categories like HTML, CSS, JavaScript, performance, accessibility, SEO, and security, each with explanations, remediation guidance, and verification steps. It is for developers auditing front-end quality and for agents that need a structured rule corpus to review code or live URLs against.

## Key features

- 385 English rules across 11 active categories (HTML, CSS, JavaScript, Performance, Accessibility, SEO, Security, Images, Testing, Privacy, Internationalization).
- A hosted MCP server exposing 11 tools such as `review_code`, `search_rules`, `audit_url`, `get_workflow`, and `get_checklist_rules` for agent-driven review.
- Rule pages with explanations, remediation guidance, and verification steps, plus a priority legend (Critical, High, Medium, Low).
- Multiple workflows: browse and filter on the website, work through the checklist in the README, or connect an MCP-capable agent to the same rule corpus.
- Installable skills (`npx skills add frontendchecklist/skills`) for reusable audit workflows and focused rule-specific guidance.

## Tech stack

- Rules authored in MDX (the repository's primary language).
- A Turborepo monorepo with a Next.js-based web app under `apps/web` and packages for the MCP server, CLI, crawler, and emails.
- Biome for linting and formatting; pnpm (`10.33.0`) as the package manager; Node engine constrained to `>=24.15.0 <25`.
- TypeScript tooling via tsx, remark/unified and remark-mdx for processing MDX rules, gray-matter for frontmatter, and lefthook for git hooks.
- Playwright for end-to-end tests, plus an MCP stdio server for local/editor integration at `packages/mcp/src/cli.ts`.

## When to reach for it

- Auditing a page, component, or pull request for accessibility, performance, SEO, and security quality.
- Pointing an MCP-capable agent at pasted code or a public URL to get rule-grounded findings.
- Running launch, accessibility, or security audits from a shared, browsable rule corpus.
- Standardizing front-end review expectations across a team.

## When *not* to reach for it

- Your project is backend or non-web; the corpus is front-end specific.
- You want a drop-in linter that fails CI automatically; this is a rule corpus and review workflow, not an enforcement tool out of the box.
- Reuse or redistribution matters to you: no SPDX license is declared in the repository metadata.

## Maturity signal

Created in 2017, this is a long-established project with a very large star count, and it has recently been rebuilt into a monorepo with an MCP server and installable skills — a modernization aimed at AI-agent workflows on top of a mature checklist. The most recent push is about a month before the snapshot and the open-issue count is very low, both consistent with an actively maintained, well-tended project. The main caveat is the absence of a declared license in the metadata, which affects how freely the corpus can be reused.

## Alternatives

- Lighthouse — use it instead when you want automated, scored audits of a running page rather than a human/agent-driven checklist.
- axe DevTools (axe-core) — use it instead when you want automated accessibility testing integrated into your test suite or browser.
- The A11y Project checklist — use it instead when you specifically want an accessibility-focused reference checklist.

## Notes

The project is explicitly positioned for "humans and AI agents," and its README checklist block is generated from the rule corpus by a `generate:readme` script rather than hand-maintained. A companion project, UX Patterns for Devs, is recommended for choosing which interface pattern to build before using this checklist to verify implementation quality.

## Tags

frontend, web-development, checklist, accessibility, performance, seo, mcp, mdx, reference, ai-agents
