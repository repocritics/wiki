# avelino/awesome-go

The canonical curated list of Go frameworks, libraries, and tooling — companion site at awesome-go.com makes the catalogue browsable.

## What it is

A community-maintained awesome-list for the Go ecosystem, sorted by domain (web frameworks, databases, network, CLI, observability, machine learning, image processing, etc.). Mirrored at awesome-go.com which adds search and category filtering on top of the README content. Maintained by Thiago Avelino and a network of category captains who shepherd PRs.

## Key features

- Comprehensive Go ecosystem coverage spanning standard application categories plus Go-specific buckets (Goroutines, Codecs, Date Pickers).
- Companion site at awesome-go.com that renders the README into a faceted, searchable directory.
- Hacktoberfest participation — a regular onboarding path for first-time OSS contributors.
- Curated, not just listed — entries can be removed when projects abandon maintenance.
- MIT-licensed.

## Tech stack

- Markdown content (the README is the directory).
- The Go language tag reflects the entries, not in-repo code.
- awesome-go.com is a separate site that consumes the README structure.

## When to reach for it

- You're starting a Go project and want a curated short-list before pip-equivalent (`go get`) installation.
- You're surveying the Go ecosystem from outside and want a vetted snapshot.
- You're building a meta-tool (AI agent, package recommender) that needs a clean Go catalog.

## When *not* to reach for it

- You want depth — entries are one-line summaries, you'll bounce out to read the project.
- You want benchmark data or active-maintenance status surfaced inline. The directory doesn't carry that.
- You want extremely recent additions — curated lists trail bleeding-edge by weeks.

## Maturity signal

174k stars, 13k forks, MIT, last push 2026-05-30. 11-year-old project with active category-captain stewardship and Hacktoberfest onboarding. Open-issues count of 191 is reasonable for a list of this size, suggesting effective triage. The awesome-go.com companion site signals sustained investment beyond just markdown editing.

## Alternatives

- `sindresorhus/awesome` — use when you want a meta-index of all awesome-lists.
- `vinta/awesome-python` — use for Python instead of Go.
- pkg.go.dev — use when you want live module data with download counts and security signals.

## Notes

The MIT license + companion site combination makes this one of the more re-indexable awesome-lists; downstream tools can train or cluster against it without licensing friction. The "category captain" model — domain experts maintain individual sections — is the durability mechanism that has kept this list relevant past most awesome-list half-lives.

## Tags

awesome-list, golang, curated-directory, library, web-framework, open-source
