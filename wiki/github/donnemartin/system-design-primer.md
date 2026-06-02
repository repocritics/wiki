# donnemartin/system-design-primer

An organized, link-heavy primer on large-scale system design — interview prep plus general engineering reading.

## What it is

A long-form, single-author study guide for system design topics, structured around the questions that typically come up in tech-company system-design interviews (Amazon, Google, Meta, etc.). Built as a flat tree of markdown sections that mix narrative explanation, diagrams, and curated outbound links. Companion Anki flashcards reinforce the material. Translated (or being translated) into 18+ languages via community PRs.

## Key features

- Coverage that spans concrete interview topics (URL shortener, social feed, key-value store, search typeahead, scaling Twitter) and the underlying primitives (CAP theorem, consistency patterns, caching strategies, async messaging, CDN).
- Anki deck companion for spaced-repetition review.
- 18+ language translation tracks in flight, each as a sibling README in the repo.
- Diagrams hosted alongside the text — both architectural sketches and queue/storage patterns.
- Self-contained — readers don't need an external course or course-provider account.

## Tech stack

- Markdown content + embedded images.
- No code build or package manifest; Python is the listed primary language because of the Anki flashcard generation scripts.
- Translation tracks are sibling files maintained via PR.

## When to reach for it

- You're preparing for a system-design interview and want one document to anchor your study around.
- You want a shared reference for team discussions ("look at the primer's section on caching strategies").
- You're a self-learner who likes long-form, link-driven study material over video.

## When *not* to reach for it

- You want production runbooks — this is a learning resource, not an operations manual.
- You want recent material on a specific new tech (LLM serving infra, vector DBs). Coverage skews to classical patterns.
- You want a license-clean derivative. `NOASSERTION` on the GitHub side; verify before redistributing.

## Maturity signal

351k stars, 56k forks, last push March 2026 — actively maintained though not constantly. 9-year-old project with sustained community contributions; 18+ in-flight translations are a strong signal of utility outside English-speaking dev circles. 549 open issues read elevated, but on closer look most are translation tracking issues rather than bug reports.

## Alternatives

- `karanpratapsingh/system-design` — use when you want a more recent, video-paired survey.
- "Designing Data-Intensive Applications" (Kleppmann) — use when you want a single coherent book-length treatment.
- `donnemartin/awesome-aws` (same author) — use when you want AWS-specific tooling instead of generic patterns.

## Notes

The project is famously the work of one prolific contributor (Donne Martin) plus translation contributors; depth varies by section because the author's expertise (and time) varies. The diagrams have been re-used in countless conference talks and engineering blog posts — citation hygiene downstream is uneven. License absence is the recurring gotcha for educational-channel reuse.

## Tags

system-design, awesome-list, education, interview-prep, scalability, distributed-systems, learn-to-code, anki, multilingual, software-architecture
