# anomalyco/opencode

A TypeScript open-source coding agent positioning itself at opencode.ai. Audit-script signals warrant a hands-on review before integration.

## What it is

A TypeScript-based coding-agent CLI distributed at opencode.ai. The README headline is "The open source coding agent" — minimal positioning text in the excerpt. Default branch is `dev` (unusual for a stable-positioned project at this star count). Created in April 2025, the project crossed 168k stars in about a year, with a notably high open-issues count (6,400+) relative to its watcher count (~640).

## Key features

- TypeScript implementation, distributed via the opencode.ai surface.
- Open-source positioning at MIT license.
- Active push cadence at generation time.

## Tech stack

- TypeScript primary.
- Default branch `dev` — operational signal that the project ships on a development branch rather than a settled `main`.

## When to reach for it

- You're cataloguing the coding-agent space and want hands-on exposure to a TypeScript implementation.
- You're explicitly evaluating opencode.ai as a hosted-or-CLI option and want the source to read before committing.

## When *not* to reach for it

- You need a vendor-supported coding agent — choose first-party Claude Code, OpenAI Codex CLI, or Cursor.
- You're risk-averse about installing third-party agent code into a credentialed loop without a thorough source review.
- The audit signals matter to you: 168k stars in 12 months, high issue-to-watcher ratio, `dev` default branch, and brand-name overlap with the broader "OpenCode" ecosystem all warrant verification before integration.

## Maturity signal

The combination of 168k stars in ~12 months, the 6,400+ open-issues count vs. ~640 watchers, and the `dev` default branch are unusual for an organically-grown OSS project. None is conclusive on its own; together they place this firmly in the "review the actual code and the maintainer history before depending" bucket. MIT license is clean. The 20k fork count is high, but in the agent space forks often correlate with hype rather than active downstream use.

## Alternatives

- `anthropics/claude-code` — use for the first-party Claude coding agent.
- OpenAI Codex CLI — use for the first-party OpenAI coding agent.
- `obra/superpowers`, `affaan-m/ECC` — comparable third-party agent / skills frameworks with overlapping target audience.
- Cursor, Windsurf — use when you want a GUI coding-agent experience.

## Notes

The "OpenCode" name has been adopted by multiple projects in the coding-agent space (including the well-known `sst/opencode`); verify which `opencode.ai` you're evaluating before installing. Star count is a poor quality signal in this category right now — coordinated stargazing campaigns are common — so read the source, the recent merged PRs, and the maintainer's other repos before integrating. License (MIT) doesn't substitute for supply-chain due diligence.

## Tags

artificial-intelligence, large-language-model, agent, typescript, command-line-interface, developer-tools, coding-agent
