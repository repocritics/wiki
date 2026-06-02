# EbookFoundation/free-programming-books

A multilingual directory of free programming books, cheat sheets, online courses, podcasts, and problem sets — maintained by the Free Ebook Foundation.

## What it is

A community-curated index of free programming learning material, sliced by both subject and human language. Originally a clone of a Stack Overflow list, the directory has been hosted on GitHub since 2013 and is now operated by the Free Ebook Foundation, a US 501(c)(3) nonprofit. Content lives across many markdown files — one per language and category — with the root README acting as a navigation hub. A separate search site (`ebookfoundation.github.io/free-programming-books-search`) indexes the entries for direct lookup.

## Key features

- Books indexed by both subject and programming language, with separate per-natural-language directories (40+ languages including English, Arabic, Chinese, French, German, Hindi, Japanese, Korean, Portuguese, Russian, Spanish, Vietnamese, and many smaller-population languages).
- Adjacent resource categories: cheat sheets, free online courses, interactive tutorials, podcasts, programming-playgrounds, problem sets, and writing guides — all grouped per language where applicable.
- Companion static + dynamic search site for direct-lookup queries instead of README-scrolling.
- Hacktoberfest-participating with explicit "good first issue" and "help wanted" labels for newcomer onboarding.
- CC-BY-4.0 license, so redistribution and indexing (including by AI corpora) is allowed with attribution.
- Documented contribution flow (`CONTRIBUTING.md`, `HOWTO.md`, `CODE_OF_CONDUCT.md`) including a translated Code of Conduct.

## Tech stack

- Markdown content end-to-end; navigation is plain bullet lists and per-language section files.
- Python language tag covers contribution-validation scripts that run in CI to enforce list formatting.
- Static-site companion via GitHub Pages (`ebookfoundation.github.io/free-programming-books/`).
- Dynamic search site is a separate sister repo.

## When to reach for it

- You want free, license-clear learning material in a specific human language for a specific programming topic.
- You're stocking a community library, school resource list, or AI training corpus with vetted free books.
- You're a contributor looking for a friendly first-time OSS PR (well-labeled good-first-issues, clear guides).

## When *not* to reach for it

- You want paid or premium courses — the entire scope is free.
- You want fresh, dated content — many entries are classic books that haven't moved in years.
- You want unified search inside the README itself — that experience lives on the companion search site, not in-repo.

## Maturity signal

389k stars, 66k forks, last push May 30 2026 — actively maintained. 12-year history under nonprofit stewardship, a tax-deductible donation path, and a structured contributor flow signal long-term institutional intent. The relatively low open-issues count (83) against the massive surface area suggests effective triage by a sustained maintainer pool.

## Alternatives

- `sindresorhus/awesome` — use when you want the meta-index of all awesome-lists rather than free books specifically.
- `freeCodeCamp/freeCodeCamp` — use when you want a paced, certifying curriculum instead of a book catalog.
- `ossu/computer-science` — use when you want a single recommended curriculum to follow end-to-end.

## Notes

The per-language directory structure is unusually deep — useful for non-English readers, but it means tooling that just scrapes the root README sees only the navigation hub, not the actual book listings. CC-BY-4.0 (with attribution) is more permissive than the typical "no license" awesome-list default; downstream re-indexers should still preserve the EbookFoundation credit. The Free Ebook Foundation's nonprofit status differentiates this from APILayer-style commercially-stewarded lists.

## Tags

awesome-list, education, books, learn-to-code, free, multilingual, nonprofit, creative-commons
