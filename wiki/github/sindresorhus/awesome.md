# sindresorhus/awesome

The canonical meta-index of "awesome lists" — curated topic lists themselves curated into one root.

## What it is

A list of lists. Each entry points to a topic-specific `awesome-*` repo (Node.js, React Native, Rust, security, databases, etc.). Maintained by Sindre Sorhus and a network of co-maintainers, it functions as the entry point for the awesome-list ecosystem: a reader who wants "what's awesome in X" lands here, follows the link to the topic list, then drills into individual projects from there.

## Key features

- 27 top-level categories spanning platforms, languages, front-end, back-end, computer science, theory, books, gaming, databases, security, hardware, networking, and more.
- Every entry links out to its own dedicated awesome-list repo — this README never duplicates a topic list's content.
- Strict contribution guidelines (`awesome.md`, `contributing.md`, `create-list.md`) define what qualifies as "awesome" and how to add or create new lists.
- CC0-1.0 licensed, so individual lists and pointers can be freely copied, mirrored, or re-indexed by downstream tools and search engines.
- Sub-categorization via nested bullets keeps related topics together (e.g. JavaScript → Promises, ESLint, Functional Programming).

## Tech stack

- Markdown only — the entire list is a single `README.md` plus three short policy documents.
- No build tooling, package manifest, or generator.
- PRs go through the contribution guide checklist before merge.

## When to reach for it

- You don't know which awesome-list to start with for a given topic.
- You want a license-clean starting point for building a derivative directory or search index over OSS topics.
- You're evaluating ecosystems ("what does the open-source landscape for Elixir look like?") and want a community-vetted entry point.

## When *not* to reach for it

- You want depth on a specific topic — you'll bounce to a sub-list within one click anyway.
- You want fresh news or trending projects. Awesome lists optimize for canonical resources, not recency.
- You're looking for evaluations or comparisons. Entries are flat one-liners — there's no scoring, no "best in class" call-out.

## Maturity signal

472k stars, 35k forks, last push May 2026 — actively curated for over a decade. CC0-1.0 with explicit contribution and creation guides signals an institutional list, not a personal scratchpad. Open-issues count near 80 is low for a list of this size and follower count, which suggests maintainers triage steadily.

## Alternatives

- `codecrafters-io/build-your-own-x` — use when you want hands-on implementation tutorials rather than topic indices.
- Topic-specific awesome lists directly — use when you already know your domain and want depth, not breadth.
- `ossu/computer-science` — use when you want a structured curriculum, not a directory.

## Notes

The README's top section is heavily sponsor-themed (Supercharge, Depot, Circleback) but does not editorialize entries — sponsorship and inclusion are separated. CC0-1.0 license is rare for a list of this scale; most awesome-lists ship under non-licensed README defaults, which makes this one safer to redistribute or train against.

## Tags

awesome-list, meta-list, curated-directory, open-source, creative-commons
