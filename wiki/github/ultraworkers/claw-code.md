# ultraworkers/claw-code

A high-velocity coding-agent project written in Rust on top of the `oh-my-codex` runtime. Star count is unusually elevated relative to project age and should not be read as a quality signal.

## What it is

Per the README, a coding-agent / CLI tool implemented in Rust against the `oh-my-codex` substrate. Repo description prominently markets the star count itself ("the fastest repo in history to surpass 100K stars") rather than the product, and emphasizes a Discord community. Project workspace lives under `rust/` per the README link structure. The actual feature surface and stability characteristics require reviewing the workspace source rather than the README.

## Key features

- Rust workspace as the implementation substrate.
- Built on top of `oh-my-codex`, per the description.
- Discord community as the primary support channel.
- MIT-licensed.

## Tech stack

- Rust at the source level; `rust/` subdirectory holds the workspace.
- `oh-my-codex` upstream dependency.

## When to reach for it

- You're explicitly experimenting with the `oh-my-codex` ecosystem and want a downstream project to study.
- You want a Rust-implemented coding agent and are willing to audit a 2-month-old project before integrating.

## When *not* to reach for it

- You need a stable, broadly-vetted coding agent — choose a first-party agent (Claude Code, Codex, Cursor) or a project with a longer track record.
- You're risk-averse about installing tools that drive credentialed agent loops.
- Star-velocity is part of your quality heuristic — this repo gained 100k+ stars in roughly two months with descriptive copy that markets the velocity itself, a pattern that warrants extra scrutiny.

## Maturity signal

193k stars accumulated over roughly two months puts this at the extreme tail of star-velocity for an OSS repo (~3,000 stars per day at average). The owner account is young, the repo's description leans on the star count as the marketing message rather than on the product, and the topics list is empty. None of these are conclusive on their own, but together they place this firmly in the "review the actual code before depending" bucket. MIT license clean. Last push 2026-05-29.

## Alternatives

- First-party `anthropics/claude-code`, OpenAI Codex CLI — use for vendor-supported, broadly-vetted agent runtimes.
- `obra/superpowers` — comparable agent-skills framework with a longer track record (October 2025 origin vs. March 2026 here).
- Plain CLAUDE.md / AGENTS.md project conventions — use when you want zero third-party dependency in your agent loop.

## Notes

Repos that hit mega-star tier on this timeline almost always rely on coordinated stargazing campaigns rather than organic adoption — the audit flag isn't a verdict, but the asymmetry between "100k stars in 2 months" and "no topics, no production references in the README's lede" should drive a hands-on review of the actual `rust/` workspace before any integration that exposes filesystem, network, or credential surface. Anyone re-indexing the GitHub OSS corpus should down-weight star count as a quality signal for this entry.

## Tags

rust, command-line-interface, agent, developer-tools, oh-my-codex
