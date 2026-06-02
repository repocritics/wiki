# mattpocock/skills

A personally-curated skills bundle published from Matt Pocock's `~/.claude/` directory — TypeScript-flavored skills for engineers using Claude Code and adjacent agent runtimes.

## What it is

Matt Pocock's published skills directory — the actual contents of his `~/.claude/` agent-skills folder, surfaced as a public repo so others can install or fork. Skills here reflect Matt's day-to-day TypeScript / type-system / library-author workflows. Distinct from the official `anthropics/skills` (first-party) and the broader third-party frameworks (`obra/superpowers`, `affaan-m/ECC`) — this is the "respected engineer publishes their setup" lane.

## Key features

- Shell-script-based skills installable into a Claude Code-shaped `.claude/` directory.
- TypeScript-flavored — Matt's domain is TS library authorship, type-system tricks, dev tooling.
- Personal-but-public — the README pitches them as "Skills for Real Engineers. Straight from my .claude directory."
- MIT-licensed.

## Tech stack

- Shell scripts as the skill substrate.
- Skills follow the SKILL.md / supporting-files convention of the broader agent-skills ecosystem.

## When to reach for it

- You're a TypeScript engineer using Claude Code and want a tested third-party skills baseline.
- You follow Matt Pocock's content (TS tips, Total TypeScript) and want his agent-side practices to match.
- You're researching the "respected engineer's skills folder" pattern as a contribution to the agent-skills ecosystem.

## When *not* to reach for it

- You want a vendor-supported skills catalog — `anthropics/skills` is the first-party alternative.
- You want non-TypeScript-flavored skills — Matt's setup is opinionated toward his stack.
- You're risk-averse about installing third-party skills into a credentialed agent loop without source review. (Even from a trusted author, audit the shell content before installation.)

## Maturity signal

114k stars accumulated since February 2026 (about 4 months at generation time) — fast-rising. The project is fundamentally Matt Pocock's personal directory published in public, so its trajectory depends on his attention. The 52 open-issues count is unusually low and reflects the personal nature of the project — issues are mostly feature suggestions. MIT license clean. Star velocity alone in the agent-skills space isn't a quality signal; the relevant signal here is "do you trust Matt Pocock's TypeScript practices?" — for many engineers, yes.

## Alternatives

- `anthropics/skills` — first-party Anthropic skills.
- `obra/superpowers` — broader, framework-style skills.
- `affaan-m/ECC` — comparable third-party agent-harness with skills, instincts, and memory layers.
- Per-project AGENTS.md / CLAUDE.md conventions — use when you want zero third-party dependency.

## Notes

The "respected engineer publishes their dotfiles-equivalent" pattern is new in the agent-skills space and Matt Pocock is one of the early high-profile examples. Anyone evaluating this should weight "do you want Matt's specific TypeScript-flavored opinions?" against "do you want a more generic skills baseline?" Audit the shell content before installing into an agent loop with filesystem or network access — even with MIT license and trusted authorship, supply-chain hygiene matters.

## Tags

artificial-intelligence, agent, claude, agent-skills, shell, developer-tools, typescript, claude-code, framework
