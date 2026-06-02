# anuraghazra/github-readme-stats

A serverless service that generates dynamic SVG stat cards for GitHub profile READMEs — embed your stars, contribution graphs, top languages, and PR streaks.

## What it is

A Vercel-hosted serverless function that renders SVG images on the fly with a user's GitHub stats: stars, total contributions, top languages, PRs, issues. Embeddable in any Markdown via image URL. Drove the popularization of "GitHub profile READMEs" — the per-user repo named after the username that GitHub renders at the top of the profile page.

## Key features

- Dynamic SVG generation for: overall stats, top languages, contributions, PRs, repository cards, and more.
- Themed presets (dark, radical, dracula, gruvbox, etc.) plus customization parameters.
- Self-hostable for higher rate limits.
- Many "card" variants — pin a repo as a card, language usage bar, etc.
- MIT-licensed.

## Tech stack

- JavaScript primary.
- Node.js serverless functions on Vercel.
- GitHub REST + GraphQL APIs for data fetching.

## When to reach for it

- You're customizing your GitHub profile README and want dynamic stat cards.
- You're a content creator showing GitHub presence on a personal site.

## When *not* to reach for it

- You want real-time, sub-minute updates — the service is cache-driven for rate-limit reasons.
- You don't want a third-party dependency in your README — self-host for sovereignty.

## Maturity signal

79k stars, 34k forks, MIT, actively maintained. The 34k forks reflect self-hosting deployments. Open-issues count of 285 tracks theme requests + rate-limit reports.

## Alternatives

- Self-hosted alternatives via the same code.
- Other profile-README enhancers (`vercel/og-image`, custom serverless functions).

## Tags

javascript, github, serverless, readme, profile, svg, vercel, mit-license
