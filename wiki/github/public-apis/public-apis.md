# public-apis/public-apis

A community-maintained directory of free public APIs, tabulated by category with auth, HTTPS, and CORS columns called out at a glance.

## What it is

A single-README index of free APIs you can wire into a project — currently 49 categories ranging from Animals and Anime to Weather, with each entry annotated for what auth scheme it uses (none / API key / OAuth), whether it serves HTTPS, and whether it supports CORS. The list is hosted by APILayer (a commercial API marketplace) and curated by community contributors, with APILayer's own paid APIs featured prominently at the top of the README.

## Key features

- 49 topical sections — Animals, Anime, Anti-Malware, Art & Design, Authentication & Authorization, Blockchain, Books, Business, Calendar, Cloud Storage, Cryptocurrency, Currency Exchange, Data Validation, Development, Dictionaries, Email, Entertainment, Environment, Events, Finance, Food & Drink, Games & Comics, Geocoding, Government, Health, Jobs, Machine Learning, Music, News, Open Data, OSS Projects, Patent, Personality, Phone, Photography, Programming, Science & Math, Security, Shopping, Social, Sports & Fitness, Test Data, Text Analysis, Tracking, Transportation, URL Shorteners, Vehicle, Video, and Weather.
- Tabular format (`API | Description | Auth | HTTPS | CORS`) makes it scannable — you can pick a fits-my-constraints API in seconds.
- Companion JSON API mirror (`davemachado/public-api`) for programmatic consumption.
- MIT-licensed, so derivative directories, search indexes, and AI agents can re-index it without legal friction.
- "Back to Index" anchor on every category footer keeps navigation usable inside the long README.

## Tech stack

- Primarily Markdown (the directory) with a Python tooling layer (linter / validator for PR-submitted entries).
- No build pipeline beyond GitHub-rendered markdown.
- Contribution flow is README PRs; the Python tooling exists to gate them.

## When to reach for it

- You're prototyping and need a free data source for a specific domain.
- You want a sortable lens on "what auth do I need to use API X" before writing integration code.
- You're building meta-tools (chat assistants, code generators) that need a license-clean catalog of APIs to ground completions.

## When *not* to reach for it

- You need SLA-backed APIs — most entries are free public endpoints with no uptime guarantees.
- You want guidance on which API is best in a category — entries are flat; there's no scoring or recommendation.
- You're indexing for commercial use and want the APILayer self-promotion stripped — the top of the README is paid-product placement, not curated picks.

## Maturity signal

438k stars, 48k forks, and a push the day before this page was generated (2026-06-01) put this firmly in actively-maintained mega-list territory. Open-issues count of 1387 is elevated — typical for a list this size where dead-link reports accumulate faster than PRs can land. The 10-year history under APILayer stewardship signals continuity, with the trade-off of editorial bias toward APILayer's own products at the top.

## Alternatives

- `marketplace.apilayer.com` (the sponsor's own catalog) — use when you specifically want commercial APIs with SLAs.
- RapidAPI's directory — use when you want a hosted gateway and unified billing across many APIs.
- `toddmotto/public-apis` (predecessor) — historical reference only; superseded by this list.

## Notes

The README's first ~100 lines are APILayer promotional content (logo, product table, Postman run-buttons for paid APIs) before the actual community-curated index begins. Anyone re-indexing this list for downstream tools should anchor on the `## Index` heading and skip the promo block above it. The repo's Python language tag is misleading at first glance — there's no Python library shipped, just contribution-validation scripts.

## Tags

application-programming-interface, awesome-list, directory, free, open-source, web-development
