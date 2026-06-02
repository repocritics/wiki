# getify/You-Dont-Know-JS

The source markdown for Kyle Simpson's *You Don't Know JS Yet* book series — a deep dive into the JavaScript language as it actually works, not as the average tutorial describes it.

## What it is

A book series originally published as You Don't Know JS (1st edition) and now in its 2nd edition under the title *You Don't Know JS Yet*. The repo hosts the in-progress and shipped book manuscripts directly as markdown — readers can study the source for free in the repo, or buy the printed/ePub editions on Leanpub and Amazon. Coverage walks JavaScript from scope/closures, asynchrony, prototypes, types, ES6 features, through the runtime model that most "intro to JS" courses gloss over.

## Key features

- Free-online reading of the manuscript content in the repo, even for books that are also sold commercially.
- Per-book directories with their own manuscript markdown, images, and code snippets.
- Coverage focused on language mechanics: scope, closures, this, prototypes, types, async, generators — the parts that bite intermediate JS developers.
- Author has been refactoring earlier editions into the 2nd-ed structure; some 1st-ed material remains for historical reference.
- Translation tracks in community forks (not in this repo directly).

## Tech stack

- Markdown manuscripts; no code project.
- Default branch is `2nd-ed`, signalling that the 2nd-edition rewrite is the canonical reference.

## When to reach for it

- You've been writing JavaScript for a while and want to fix the parts you've been guessing about (closures, `this` binding, the asynchrony model).
- You're a tech-lead reviewing code in a codebase where "it works but I don't know why" comments are common — these books are the canonical pointer.
- You're preparing for senior-IC JS interviews where language internals come up.

## When *not* to reach for it

- You're new to programming — start with a beginner course first; the books assume comfort with the basics.
- You want framework material (React, Vue, Node patterns) — the series is language-focused, not framework-focused.
- You want a license-clean training corpus — `NOASSERTION` SPDX; verify before commercial reuse.

## Maturity signal

184k stars, 33k forks, last push February 2026 — active, in the long-tail "occasional polish" cadence that fits a book series rather than a software project. 13-year-old reference now in its 2nd edition. Open-issues count of 2 reflects the author's preferred workflow (PRs and discussion, not issue triage). The free-source-of-paid-book model is unusually trust-building.

## Alternatives

- Eloquent JavaScript (Marijn Haverbeke) — use when you want a one-volume tutorial covering language + browser APIs.
- JavaScript: The Definitive Guide (Flanagan) — use when you want a reference-style treatment.
- MDN JavaScript reference — use when you need lookups during day-to-day coding.

## Notes

License (`NOASSERTION`) is the recurring gotcha — the books themselves are sold commercially under specific terms; the in-repo manuscript content is governed by the LICENSE file in the working tree, which downstream consumers should pin to. The 1st-edition material is in subdirectories on the same default branch and is being progressively superseded; readers should default to the 2nd-edition content unless they have a specific historical reason.

## Tags

awesome-list, javascript, books, education, learn-to-code, language-fundamentals
