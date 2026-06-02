# awesome-selfhosted/awesome-selfhosted

A free/open-source-only directory of network services and web applications you can host on your own server instead of consuming as SaaS.

## What it is

A curated list of self-hostable alternatives to mainstream cloud services — analytics platforms, password managers, file sync, photo libraries, IRC/chat servers, project management, productivity suites, and dozens of other categories. Strict editorial scope: only Free Software entries are included; commercial / source-available / closed-source projects live on a separate `non-free.md` page. Mirrored at awesome-selfhosted.net as an HTML site that's the recommended browsing surface.

## Key features

- Free-only editorial line — entries that aren't Free Software per the GNU definition get filtered to a sibling list.
- Automated CI checks against the data repo to flag dead links and unmaintained projects.
- Companion HTML site with search, filtering, and license badges, generated from the same data.
- Two-repo split: this list is human-readable; sister `awesome-selfhosted-data` repo holds the structured source-of-truth and CI workflows.
- Sustainable via Liberapay donations rather than corporate sponsorship.

## Tech stack

- Markdown content, automated link/maintenance checks via GitHub Actions in the data repo.
- No code in this repo — Python language tag is `null` because this is content-only.
- HTML site is built from `awesome-selfhosted-data` separately.

## When to reach for it

- You want to replace a SaaS service with a self-hosted equivalent and need a vetted starting point.
- You're standing up a personal cloud / home server and want a curated catalog of what's worth installing.
- You're a privacy- or sovereignty-focused user who wants to keep data on your own hardware.

## When *not* to reach for it

- You're not willing to run a server — every entry assumes self-hosting capability.
- You want to use closed-source alternatives — those are explicitly filtered out (see the non-free sibling list).
- You want SLAs or commercial support — entries are community-maintained Free Software with no uptime guarantees.

## Maturity signal

296k stars, 13k forks, open-issues count of 0 — the cleanest issue tracker among mega-lists, signalling both the two-repo split (issues live in the data repo) and rigorous maintenance. Last push the day before generation (2026-05-31). 11-year-old project with Liberapay funding indicates a sustained, community-funded operation rather than a single-author burnout risk.

## Alternatives

- `sindresorhus/awesome` — use when you want the meta-list of all awesome-lists.
- `Kickball/awesome-selfhosted` (original predecessor) — historical reference; this is the active fork.
- `freedomofpress/dangerzone` and the broader privacy-tools ecosystem — use when your concern is specifically threat-model-driven.

## Notes

The strict Free Software editorial line is unusual in the awesome-list space; it's a feature, not a bug, for users whose concerns are licensing and data sovereignty. The split between this repo (human-readable) and the data repo (machine-readable structured source) is well-engineered — any downstream tooling should pin to the data repo's JSON/YAML, not parse the markdown here. License is `NOASSERTION`; downstream re-indexers should check the LICENSE file directly.

## Tags

awesome-list, self-hosted, open-source, privacy, curated-directory, sovereignty, free-software
