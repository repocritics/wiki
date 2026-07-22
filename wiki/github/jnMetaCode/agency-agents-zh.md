# jnMetaCode/agency-agents-zh

A Chinese-language collection of plug-and-play AI expert role definitions that install into Claude Code, Cursor, Copilot, and 18 other agent tools.

## What it is

agency-agents-zh is a community Chinese edition of the agency-agents role library: a set of Markdown files where each file defines an AI "expert" persona with its own identity, rules, workflow, and deliverables, organized across roughly 20 departments (engineering, design, marketing, finance, game development, GIS, and more). It is for people using AI coding and agent tools who want ready-made specialist prompts rather than writing them from scratch. Beyond translating the upstream English project, it adds original agents built for the Chinese market — platform operations for Xiaohongshu, Douyin, WeChat, Bilibili, Feishu, and DingTalk, plus cross-border e-commerce, government, healthcare compliance, and other verticals.

## Key features

- 268 agent role definitions across about 20 departments, each specifying identity, key rules, workflow, and expected deliverables rather than a one-line "you are an expert" prompt.
- Per the project's own count: 215 agents translated from upstream plus 53 original China-market agents.
- One-command install to 18 AI tools via `scripts/install.sh`, with `scripts/convert.sh` handling format conversion for tools that need it.
- OpenClaw integration splits each agent into `SOUL.md` (persona), `AGENTS.md` (capabilities), and `IDENTITY.md` (summary) for multi-agent orchestration.
- Pairs with a separate orchestrator, agency-orchestrator, that runs multiple agents in a DAG from a single natural-language request.
- An online gallery lets you search, filter by department, and copy each agent's full prompt without installing anything.

## Tech stack

- Primary language Shell — the install and convert scripts under `scripts/`.
- The payload is content (Markdown agent definitions) rather than executable software; agents are consumed by external agent tools.
- Also published as an npm package `agency-agents-zh` (version 1.2.7 in `package.json`), with a Node count-check script (`scripts/check-counts.mjs`).
- MIT license. No runtime framework or language dependency of its own.

## When to reach for it

- You use Claude Code, Cursor, Copilot, OpenClaw, or another supported tool and want ready-made specialist personas.
- You operate in the Chinese market and need agents tuned for local platforms (Xiaohongshu, Douyin, WeChat, Feishu, DingTalk) or cross-border e-commerce.
- You want a starting library of structured prompts to adapt, or to drive multi-agent DAG workflows via agency-orchestrator.

## When *not* to reach for it

- You want executable tooling — this is prompt and persona content, not a framework or runtime.
- You need English-first content; this is the Chinese edition, and the upstream agency-agents is the English source.
- You want a small, curated set — 268 roles is a broad catalog you will need to filter.
- Your agent tool does not support Agent-Skills-style loading, in which case you copy the files in manually.

## Maturity signal

Created in March 2026, the repository has roughly 18,000 stars, a recent last push (July 2026), and only 1 open issue, which fits a content library where there is little running code to file bugs against. It reads as an actively maintained, popular downstream community fork that tracks an upstream project, with breadth (departments and China-market coverage) as its main axis of growth rather than code complexity.

## Alternatives

- **agency-agents (msitarzewski/agency-agents)** — use the English upstream instead when you do not need Chinese localization or the China-market agents.
- **agency-orchestrator** — not a library of definitions but the companion runner; reach for it when you need to execute these roles as a coordinated DAG rather than only install prompt files.

## Notes

- The README stresses that these are not generic prompt templates: each agent defines how the expert thinks, works, and what it delivers (for example, the security engineer reviews code against the OWASP Top 10).
- The README front-loads several paid API-gateway sponsors, but the project content itself is MIT-licensed.
- Counts disagree slightly across sources: the description and npm package say 267, while the README and gallery say 268.

## Tags

ai-agents, prompt-engineering, system-prompt, multi-agent, claude-code, cursor, chinese, llm, agent-skills, shell
