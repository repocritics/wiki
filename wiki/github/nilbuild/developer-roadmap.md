# nilbuild/developer-roadmap

The codebase behind roadmap.sh — interactive, community-maintained roadmaps that walk through what to learn for each major developer role.

## What it is

The source for roadmap.sh, a site that publishes node-by-node interactive paths for becoming a frontend, backend, DevOps, blockchain, Go, Java, Python (and more) developer. Each roadmap is a clickable graph: tap a node to read about that topic, mark it done, drill into curated resources. The roadmaps are community-maintained; PRs against the data drive the site.

## Key features

- 20+ role-specific roadmaps (frontend, backend, DevOps, DevSecOps, full-stack, blockchain, QA, Android, iOS, Go, Java, Python, C++, React, Vue, Angular, JavaScript, Node.js, software architect, DBA, and more).
- Interactive graph UI — every node is a clickable summary that opens to a topic page with recommended resources.
- Companion "Best Practices" and "Questions" sub-sites layered on top of the same content model.
- YouTube channel reuses the same node graphs for explanatory video content.
- Multi-language translations of the README and select content.

## Tech stack

- TypeScript primary.
- The roadmap.sh site is a Next.js / Astro–style web app driven by a structured content tree under `public/` and `src/`.
- No explicit manifest surfaced in the prompt's tree slice — the project's build tooling lives deeper in the repo.

## When to reach for it

- You're new to a role (frontend, DevOps, etc.) and want a sequenced "what should I learn next" view rather than a flat resource list.
- You're mentoring junior engineers and want a shared map to point at when their next step isn't obvious.
- You want a curated, drill-down topic graph rather than a "100 links" markdown list.

## When *not* to reach for it

- You want depth — each node is a launchpad, not a full tutorial.
- You want personalization. The roadmaps are role-shaped, not skill-shaped; your specific gap may not map onto a node.
- You want commercial usability without checking — `NOASSERTION` license on the GitHub side; verify the LICENSE file before redistributing content.

## Maturity signal

355k stars, 44k forks, last push the day before generation (2026-06-01) — actively maintained and one of the most-followed educational repos on GitHub. The project predates the move to the `nilbuild` org and has been in continuous development since 2017. Open-issues count of 15 is unusually low for a repo of this size and indicates aggressive triage or that most contributions land via discussion, not GitHub issues.

## Alternatives

- `freeCodeCamp/freeCodeCamp` — use when you want a paced, exam-gated curriculum rather than a topic map.
- `ossu/computer-science` — use when you want a single recommended CS curriculum following university course order.
- The Odin Project — use when you want a hands-on, project-driven path for full-stack web.

## Notes

The repo moved organizations (originally `kamranahmedse/developer-roadmap`, later transferred under the `nilbuild` org); the underlying project and roadmap.sh site are unchanged. License is reported as `NOASSERTION` even though the project has been redistributed broadly — anyone using the roadmap content downstream should pin to the LICENSE file in the working tree at the commit they consumed.

## Tags

awesome-list, education, developer-roadmap, learning, curriculum, typescript, devops, frontend, backend, learn-to-code, interactive
