# ripienaar/free-for-dev

A curated list of SaaS, PaaS, and IaaS offerings that have free tiers useful for DevOps and developers — the canonical "what can I get for $0 to bootstrap my project" reference.

## What it is

A community-maintained directory of services that offer no-cost tiers suitable for indie developers, side projects, and DevOps tooling: monitoring, error tracking, CI/CD, package hosting, databases, CDN, DNS, email, analytics, identity, and many other categories. Each entry includes a brief description of what the free tier includes. Companion site at free-for.dev provides the polished browsing surface.

## Key features

- Coverage across DevOps and developer tool categories — APIs, monitoring, CI/CD, package hosting, databases, CDN, identity, observability, email, SMS, etc.
- Per-entry notes on what the free tier specifically includes.
- Curated removals when "free" tiers shrink or disappear.
- Companion site at free-for.dev with search and category navigation.
- Strict inclusion criteria — actually-free tiers, not trial offers or "free with caveats" gimmicks.

## Tech stack

- Markdown content as the source-of-truth.
- HTML at the language tag (companion site).
- No build tooling at the repo level beyond the site generation.

## When to reach for it

- You're standing up a side project and want to bootstrap with $0 of infrastructure cost.
- You're a CTO/founder mapping the cheapest path to a working MVP.
- You're curating an internal "approved free tools" list for a small org.

## When *not* to reach for it

- You're at scale — free tiers usually don't matter past 100+ engineers; paid plans are required.
- You want commercial recommendations with SLAs — the focus is free, not "best".
- You want a license-clean redistribution — SPDX is `null`; verify LICENSE before vendoring.

## Maturity signal

122k stars, 13k forks, last push the day this page was generated. 11-year-old project under R.I. Pienaar with strong community contribution velocity. Open-issues count of 14 is exceptionally low — the maintainer triages aggressively. Companion site (free-for.dev) is actively deployed. License absence is the typical awesome-list gotcha; downstream redistribution should preserve attribution.

## Alternatives

- `sindresorhus/awesome` — broader meta-list across topics, not free-tier-focused.
- BetterStack's free-tier list, indiehackers.com directory — use for indie-founder-focused free-tier curation.
- Google Cloud / AWS / Azure free-tier pages — use for vendor-specific authoritative info.

## Notes

The "actually free" inclusion bar is the project's defining curation choice — many lists include trials and "free with conditions" entries; this one rejects them. Updates lag actual service-pricing changes by weeks at most. Anyone planning long-term infrastructure should re-verify any specific entry's current free-tier terms before depending on it; free tiers shift more than the list can track in real time.

## Tags

awesome-list, devops, free, saas, paas, iaas, developer-tools, curated-directory
