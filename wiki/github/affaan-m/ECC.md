# affaan-m/ECC

An "agent harness performance-optimization system" — skills, instincts, memory, and security plumbing aimed at coding agents (Claude Code, Codex, OpenCode, Cursor, and others).

## What it is

A framework that the README pitches as a harness-native operator system for agentic work — it bundles skill catalogs, an "instincts" layer that gates agent decisions, a memory subsystem, and security/research-first development conventions. Designed to plug into the major coding-agent runtimes (Claude Code, Codex, OpenCode, Cursor) rather than being a standalone agent. Companion site at ecc.tools.

## Key features

- Targets multiple agent harnesses: Claude Code, Codex, OpenCode, Cursor.
- "Skills + instincts + memory + security" architecture per the README's positioning.
- Multilingual README distribution (English, Portuguese Brazil, Simplified/Traditional Chinese, Japanese, Korean, Turkish, Russian, Vietnamese, Thai, German).
- Topics declare: ai-agents, anthropic, claude, claude-code, developer-tools, llm, mcp, productivity.
- MIT-licensed.

## Tech stack

- JavaScript at the source level.
- MCP (Model Context Protocol) integration listed in the topics — fits with the multi-harness positioning.
- Companion ecc.tools site for distribution and onboarding.

## When to reach for it

- You're already running an agent harness (Claude Code, Codex, etc.) and want a maintained set of skills + memory + instincts layered on top.
- You want a multilingual community surface — translations are unusually complete for a project this age.
- You're evaluating agent-harness frameworks and want to compare against `obra/superpowers` and similar.

## When *not* to reach for it

- You want a vendor-stable, first-party agent layer — Anthropic and OpenAI ship their own skills and the third-party layer may diverge.
- You're risk-averse about installing third-party code into a credentialed agent loop without an extensive review.
- You want a minimal CLAUDE.md-style convention instead of a framework — this is the heavier end of the spectrum.

## Maturity signal

202k stars accumulated since January 2026 (about 5 months at generation time) puts this firmly in the fast-rising agent-harness wave. Last push 2026-05-31 — actively pushed. The combination of mega-velocity star growth + a young owner account + a hot category warrants skepticism: cross-check the actual skill catalog and the upstream commit history before plugging into a credentialed agent. The audit signal isn't a verdict of fake; it's a flag that the popularity-as-quality heuristic doesn't apply in this niche right now.

## Alternatives

- `obra/superpowers` — directly comparable agent-skills framework; same target audience.
- `anthropics/skills` and other first-party catalogs — use when you want vendor-supported baseline.
- Per-project AGENTS.md / CLAUDE.md conventions — use when you want zero-framework, just project-specific guidelines.

## Notes

The agent-harness layer space is highly fast-moving and prone to star-velocity pumps; star count is unreliable as a quality signal here. Anyone integrating ECC into an agent with filesystem, network, or credential access should review the JS source, the skill set, and the MCP integration's permission scopes before installation. The multilingual README distribution is unusual for a 5-month-old project and may reflect organized rollout rather than organic community translation.

## Tags

artificial-intelligence, large-language-model, agent, claude, cursor, framework, javascript, developer-tools, model-context-protocol, multilingual
