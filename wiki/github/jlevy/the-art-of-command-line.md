# jlevy/the-art-of-command-line

A single-page distilled guide to mastering the Unix command line — the "if you only read one CLI reference, read this" recommendation.

## What it is

A long, well-organized markdown document that walks through command-line essentials, intermediate workflows, and power-user techniques. Covers shell basics, job control, files and data processing, system debugging, one-liners, performance tuning, and remote work. Aimed at engineers who already use the terminal but want to plug gaps and pick up shortcuts they've been missing. Translated into 15+ languages via sibling READMEs.

## Key features

- Single-document scope — no chapters, no sequencing, just one navigable file.
- Coverage tiers: basics → everyday use → processing files and data → system debugging → one-liners → obscurity.
- Cross-platform notes (Bash on Linux, macOS, Windows via WSL).
- 15+ language translations.
- Recommendations are opinionated and brief — "use X for Y" rather than full explanations.

## Tech stack

- Markdown only.
- No build, no manifest, no code.

## When to reach for it

- You've been using the terminal for a year and feel like you're missing 80% of its potential.
- You're mentoring junior engineers and want a single reference to point at.
- You're rebuilding shell muscle memory after years in IDE-driven workflows.

## When *not* to reach for it

- You're a complete beginner — the document assumes shell familiarity and shorthand recognition.
- You want deep dives — entries are one-liners, not tutorials.
- You need a license-clean redistribution — SPDX is `null`; check the LICENSE file before reuse.

## Maturity signal

161k stars, 15k forks, last push June 2024 — in long-tail polish mode rather than active expansion. The CLI canon doesn't change much; the document ages well. Open-issues count of 254 is mostly translation tracking and minor corrections. Translation breadth (15+ languages) signals utility across non-English-speaking dev communities.

## Alternatives

- `tldr-pages/tldr` — use when you want man-page-style command examples.
- "The Linux Command Line" (book) — use when you want chapter-by-chapter pacing.
- `donnemartin/data-science-at-the-command-line` — use when you specifically want data-engineering CLI patterns.

## Notes

The single-page format is the project's defining choice — it works because the author resists scope-creep. License absence is the typical gotcha for educational repos of this vintage; redistribution should preserve attribution. The translation network is community-maintained and lags the English source by months.

## Tags

awesome-list, command-line-interface, shell, bash, linux, education, unix, learn-to-code, productivity, multilingual
