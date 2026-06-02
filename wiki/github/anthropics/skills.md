# anthropics/skills

The official Anthropic repository for Agent Skills — first-party skill bundles that ship into Claude Code and other Anthropic agent surfaces.

## What it is

Anthropic's canonical Agent Skills catalog. Skills are bundled, model-callable capabilities that agents can invoke during a session — file operations, search patterns, project conventions, workflow recipes. The repo is the public surface where the Anthropic team publishes the skills it ships with Claude Code, the Claude API harness, and Anthropic-built agents. Skills here are intended to be authoritative reference implementations; community skill repositories (third-party "skills" repos) reference these for shape.

## Key features

- Official, first-party skill implementations for Anthropic agents.
- Python primary across the skill catalog.
- Topic-tagged as `agent-skills` — the canonical anchor for the broader Skills ecosystem.
- Active push cadence — Anthropic publishes updates as new skills land in Claude Code.

## Tech stack

- Python primary.
- Skill files follow the SKILL.md / supporting-files convention used across Anthropic's agent surfaces.

## When to reach for it

- You're customizing Claude Code or another Anthropic agent and want the official skill catalog to extend from.
- You're writing third-party skills and want the canonical shape to match.
- You're researching the Skills ecosystem and need the first-party reference.

## When *not* to reach for it

- You want a community-curated skill library — try the `anthropics/skills`-adjacent third-party catalogs (e.g. `obra/superpowers`).
- You're not on an Anthropic agent surface — skills here are designed for Claude Code's harness expectations and won't drop into a generic agent runtime without adaptation.
- You want a license-clean redistributable bundle — SPDX is `null`; check the LICENSE file before commercial redistribution.

## Maturity signal

145k stars accumulated since September 2025 — fast-rising but proportional given that the Skills ecosystem itself emerged in late 2025. Last push 2026-05-29. 917 open-issues count is moderate; many are likely skill-request issues rather than defects. License absence is the recurring gotcha — Anthropic's first-party skill publishing pattern is still evolving and downstream consumers should pin to the LICENSE file at the commit they consume.

## Alternatives

- `obra/superpowers`, `affaan-m/ECC` — third-party agent-skills frameworks with overlapping target audience but different curation.
- Per-project AGENTS.md / CLAUDE.md conventions — use when you want zero-framework, just-your-project skills.
- Anthropic Claude Code's bundled defaults — use when you only need the skills that ship pre-installed.

## Notes

The "official Anthropic skills" framing is the most important non-obvious fact — third-party skill repositories cite this as canonical, so changes here propagate downstream. Anyone integrating skills into a production agent loop should pin to a specific commit rather than the moving `main` branch. License absence + Anthropic stewardship + skills ecosystem maturity together mean this is the safest first-party reference to vendor today, but verify license terms at consumption.

## Tags

artificial-intelligence, large-language-model, agent, claude, anthropic, python, agent-skills, framework
