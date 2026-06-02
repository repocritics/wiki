# freeCodeCamp/freeCodeCamp

The full codebase + curriculum behind freecodecamp.org — a donor-funded 501(c)(3) learning platform that has certified its way to a six-figure alumni hiring count.

## What it is

A monorepo containing the freeCodeCamp web platform (Node API + React client) plus the interactive curriculum content that powers six full-stack developer certifications and four language certifications. The platform is run as a nonprofit. Self-paced lessons, projects, and exams are free; certifications are recognized and verifiable. The code is BSD-3-Clause; curriculum content is separately copyrighted.

## Key features

- Six full-stack certifications (Responsive Web Design, JavaScript, Front-End Libraries, Python, Relational Databases, Back-End + APIs) plus four language certifications (A1/A2 English, A1 Spanish, A1 Chinese).
- Curriculum delivered as interactive lessons, workshops, labs, reviews, and quizzes — five required projects per cert, followed by a proctored-style exam.
- Verification: certifications are publicly linkable, revocable under the Academic Honesty Policy.
- Adjacent learning surfaces shipped from the same org: forum, YouTube channel, news publication, Discord.
- Internationalization via Crowdin (workflows for client UI + curriculum upload/download), driving the multi-language platform.
- Heavy CI surface — workflows for Playwright e2e, third-party e2e, devcontainer CI, Docker registries (DOCR + GHCR), deploy split between API and client.

## Tech stack

- TypeScript across both `api/` and the client (primary language flagged TypeScript).
- Prisma for the API data layer (two schemas: `exam-creator`, `exam-environment`).
- React on the client side (curriculum lessons are interactive React components).
- pnpm/turbo-style monorepo with husky pre-commit + prettier + stylelint.
- DevContainer-supported local environment (`.devcontainer/devcontainer.json` + docker-compose).
- Crowdin for translations; GitHub Actions for CI, deploy, label management, spam control, and PR-title fixing.

## When to reach for it

- You're learning to code from scratch and want a paced curriculum with verifiable credentials.
- You're a teacher or mentor placing students on a structured path with exam-ended modules.
- You're studying production-scale open-source education platforms (curriculum-as-data, exam-environment isolation, i18n at scale).

## When *not* to reach for it

- You want a freeform reference manual — the curriculum is sequenced and exam-gated, not browsable.
- You want short-form, topical lessons. Each certification is hours-to-weeks of work.
- You want to fork the curriculum content for redistribution; the `/curriculum` directory is separately copyrighted, not BSD-3-Clause.

## Maturity signal

445k stars, 44k forks, last push the same day this page was generated (2026-06-02) — top-tier active. 11-year-old project run as a registered nonprofit; the dual-license split (code BSD-3-Clause, curriculum copyrighted) is the kind of legal hygiene that signals long-term institutional intent. Open-issues count of 177 is low relative to the surface area, which suggests effective triage.

## Alternatives

- The Odin Project — use when you want a curriculum that's lighter on certifications and heavier on self-directed projects (also Remixed into freeCodeCamp).
- Codecademy — use when you want a polished UI with paid premium tracks; freeCodeCamp is free end-to-end.
- Coursera / edX OSS sequences — use when you want university-style courseware with synchronous cohorts.

## Notes

The `/curriculum` directory's separate copyright clause is the operational reason this is not a single-license repo, and it constrains how downstream tools (including AI training corpora) can use it. The repobeats embed in the README signals deliberate analytics on contribution patterns. Co-pilot instructions file (`.github/copilot-instructions.md`) is present, which tells you the maintainers expect AI-assisted contributions to follow specific conventions.

## Tags

education, certification, curriculum, typescript, react, prisma, learn-to-code, nonprofit, internationalization, end-to-end-testing
