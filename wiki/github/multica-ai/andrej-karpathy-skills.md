# multica-ai/andrej-karpathy-skills

A single CLAUDE.md file pitched as Andrej Karpathy's "LLM coding pitfalls" advice for Claude Code — fast-rising repo with strong audit signals warranting hands-on review before use.

## What it is

The README claim: a CLAUDE.md file that codifies Andrej Karpathy's observations on common LLM coding failure modes, intended to be dropped into a Claude Code project to "improve behavior". One file, no real product, no licensing declaration. The repo's distinguishing characteristic in the cold-start corpus is anomalous star velocity (165k stars in ~4 months under an owner account also new in that window).

## Key features

- A single CLAUDE.md file as the deliverable.
- Pitch: derived from Andrej Karpathy's observations on LLM coding pitfalls.
- Distributed by being copied into a target project's working directory.

## Tech stack

- No language tag (`null`) — this is content, not code.
- Single markdown file.

## When to reach for it

- You're cataloguing the agent-skills space and want hands-on exposure to every kind of entry, including the high-velocity-with-thin-substance variety.
- You're studying how brand-name attachment (Karpathy) plus star-velocity correlates with downstream adoption.

## When *not* to reach for it

- You're looking for actual Karpathy-authored content — verify whether this repo is endorsed or authored by Andrej Karpathy himself (his personal repos are at github.com/karpathy). Brand-name attachment in a third-party repo isn't an endorsement signal.
- You want production-ready Claude Code conventions — first-party `anthropics/skills` or per-project CLAUDE.md files written by your team are safer choices.
- You're risk-averse about installing third-party content (even non-executable markdown can shape an agent's behavior in unintended ways) without a thorough review.

## Maturity signal

165k stars in ~4 months under a 4-month-old owner account → extreme star-velocity that is uncommon for organically-grown OSS. License absent. Description is short and brand-attached. The audit script flags this with 4 simultaneous heuristics: mega-velocity, young-owner-fast-rise, low-owner-repos, and name-mimic (the repo's name claims an attachment to "Andrej Karpathy" but the owner is not in the allowlist for that brand). None of these signals individually is conclusive of bad intent — together they place this firmly in the "verify before trusting" bucket.

## Alternatives

- `anthropics/skills` — first-party Anthropic skills catalog.
- Andrej Karpathy's actual repos (`karpathy/nanoGPT`, `karpathy/llm.c`, etc.) — use when you specifically want Karpathy-authored content.
- Per-project CLAUDE.md / AGENTS.md files — use when you want zero third-party dependency.

## Notes

The "single markdown file pitched as derived from a celebrity engineer's observations" pattern is increasingly common in the agent-skills wave and should be evaluated carefully. Brand-name attachment in a repo title doesn't equal authorship or endorsement. Anyone vendoring this CLAUDE.md into a project should read it first, verify the claims against Karpathy's own public statements, and consider whether the same advice could be written from primary sources rather than relying on a third-party intermediary. License absence is the recurring gotcha for re-indexing or commercial reuse.

## Tags

claude, agent, claude-code, large-language-model, awesome-list, markdown
