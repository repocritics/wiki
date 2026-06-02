# vinta/awesome-python

An opinionated, deeply categorized index of Python frameworks, libraries, tools, and resources — the canonical entry point for "what's good in Python".

## What it is

A single curated list (mirrored at awesome-python.com as a searchable site) of Python projects organized by domain. Coverage spans AI/ML, web frameworks, data tooling, async runtimes, packaging, testing, observability, and dozens of niche categories. "Opinionated" is the right word: maintainers cull entries that don't meet quality bars, so the list functions as a quality signal as much as a directory. Sponsorship banners at the top fund maintenance.

## Key features

- Dozens of categories ranging from AI/Agents and Deep Learning to Templating, CMS, Caching, and Distributed Computing.
- Companion site at awesome-python.com adds search and filtering on top of the README content.
- Tight curation — open issues count near 20 indicates aggressive triage rather than a backlog of dead entries.
- Sponsorship slot in the README finances ongoing maintenance.
- Multilingual community via translated forks (not in this repo directly).

## Tech stack

- Markdown content end-to-end.
- The repo's Python language tag reflects entries, not in-repo code — no build tooling at the root.
- awesome-python.com is a separate site that consumes the README structure.

## When to reach for it

- You're standing up a Python project and want a curated short-list of "the good library for X" before pip-installing the first match.
- You're surveying the ecosystem after a long absence and need a "what's new and what's still standard" anchor.
- You're building meta-tools (AI agents, internal package recommenders) that need a vetted Python catalog.

## When *not* to reach for it

- You want depth — entries are one-line descriptions, you'll bounce out to read the actual project.
- You want comparison/benchmark data. The list is opinionated but not annotated with trade-off explanations.
- You want recent-news flow. Curation favors canonical entries over trend-of-the-week.

## Maturity signal

300k stars, 28k forks, last push two days before generation (2026-05-30) — actively curated. 12-year-old project, low open-issues count, and explicit sponsor funding all signal sustained maintenance. The README's claim of "#10 most-starred repo on GitHub" tracks with its position in awesome-list ecosystem.

## Alternatives

- `sindresorhus/awesome` — use when you want the meta-index of all awesome-lists, not Python-specific.
- `python/cpython` — use for the canonical Python language itself.
- PyPI search + ecosystem dashboards (libraries.io, snyk advisor) — use when you want live download counts and security signals alongside the directory.

## Notes

License declared `NOASSERTION` despite the project's broad redistribution. Anyone training models or building search indexes on top of this list should pin to the LICENSE file at the commit they consumed; the README explicitly invites contributions but doesn't spell out downstream licensing. The "opinionated" curation means entries can be controversial — a library being absent isn't necessarily proof of irrelevance.

## Tags

awesome-list, python, directory, curated-directory, open-source, machine-learning, web-framework, library
