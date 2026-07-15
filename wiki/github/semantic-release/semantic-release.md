# semantic-release/semantic-release

> Fully automated version management and package publishing, driven entirely by your commit messages.

[GitHub repo](https://github.com/semantic-release/semantic-release) ·
[Official website](https://semantic-release.org) ·
[License: MIT](https://github.com/semantic-release/semantic-release/blob/master/LICENSE)

## Overview

semantic-release removes the human from the release decision. Instead of a person choosing a version number and writing a changelog, it reads the commit messages since the last release, infers the correct [Semantic Versioning](https://semver.org) bump (patch / minor / major) from a formalized convention, then tags, publishes, and announces the release automatically. The pitch, repeated in the README, is that it "removes the immediate connection between human emotions and version numbers"[^1] — releases become an unromantic side effect of merging to a release branch rather than a ceremony.

The project has existed since 2014 and is one of the older, more established pieces of the JavaScript release-automation ecosystem[^2]. It is npm-first in its defaults but not npm-only: the core is a plugin-driven lifecycle, and the same engine publishes to any registry or git host that has a plugin. The defining tension is discipline-for-automation: semantic-release only works if every contributor writes machine-parseable commit messages (Angular / Conventional Commits by default). Teams that adopt it inherit a hard requirement on commit hygiene, usually enforced with commitlint and a PR squash-merge policy. Skip that discipline and the tool silently does the wrong thing — no release, or the wrong bump.

The second defining constraint is that it is built for CI, not the desktop. semantic-release is meant to run non-interactively on a build server after a successful test run, with registry and git credentials injected as environment variables. Running it locally is possible but off the happy path.

## Getting Started

```bash
npm install --save-dev semantic-release
```

A minimal `.releaserc.json` at the repo root:

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/npm",
    "@semantic-release/github"
  ]
}
```

Then run it from CI (here, GitHub Actions) with credentials in the environment:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0        # REQUIRED — semantic-release needs full git history
- run: npm ci
- run: npx semantic-release
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

With that in place, a merged commit like `fix: correct off-by-one in paginator` cuts a patch release; `feat: add retry option` cuts a minor; a `BREAKING CHANGE:` footer cuts a major.

## Architecture / How It Works

semantic-release is a fixed lifecycle plus a set of plugins that hook into it. On each run it walks the git tags to find the last release, collects the commits since, and asks the plugins — in order — to participate in a defined sequence of steps: `verifyConditions`, `analyzeCommits`, `verifyRelease`, `generateNotes`, `prepare`, `publish`, `addChannel`, `success`, and `fail`[^3]. The core owns none of the domain logic; every meaningful action lives in a plugin.

The default plugin chain is four packages, each maintained in its own repo under the same org:

- **@semantic-release/commit-analyzer** — parses commit messages (Angular preset by default) and returns the release type. This is the brain of the version decision.
- **@semantic-release/release-notes-generator** — turns the same commits into a formatted changelog fragment.
- **@semantic-release/npm** — bumps `package.json` and runs `npm publish`.
- **@semantic-release/github** — creates the GitHub Release, uploads assets, and comments on the issues/PRs contained in the release.

Common additions are **@semantic-release/changelog** (writes/updates `CHANGELOG.md`) and **@semantic-release/git** (commits the bumped files back to the repo). Because the whole thing is a plugin bus, publishing to GitLab, a Docker registry, or arbitrary shell commands (@semantic-release/exec) is just a matter of adding plugins — the version-decision logic never changes.

The state store is git itself. There is no database and no service; the "last release" is whatever the most recent matching git tag says. Distribution channels (npm dist-tags, prerelease lines) are modeled through the `branches` config: pushing to `next` or `beta` produces prereleases, pushing to a maintenance branch like `1.x` produces a maintenance release. This makes the release history fully reconstructable from the repo, but also means the branch configuration is load-bearing and easy to misconfigure.

## Production Notes

**Shallow clones break it.** The single most common failure is a CI checkout with default depth. semantic-release needs the full tag and commit history to find the previous release; `fetch-depth: 0` (or the equivalent) is mandatory. Without it you get spurious "no release" runs or first-release loops.

**Commit discipline is the whole ballgame.** If merged commits don't follow the convention, the analyzer sees no releasable change and exits silently with no release. The usual fix is squash-merge with an enforced PR-title convention plus commitlint on the title, so the merge commit is always well-formed regardless of the messy branch history.

**ESM-only and moving Node baselines.** Since v20 (2023) the package is ESM-only — it cannot be `require()`d, and `release.config.js` must be ESM or you must use a `.json`/`.cjs` variant[^4]. The project also tracks a moving minimum Node version and drops EOL Node lines aggressively; major bumps have repeatedly been "we raised the Node floor" more than new features. Pin your CI Node version deliberately.

**Weak monorepo story.** semantic-release is architecturally single-package: one `package.json`, one version line, one tag namespace. Monorepos with independent per-package versions are not a first-class use case. Teams either run it once per package with tag prefixes and the `semantic-release-monorepo` community wrapper, or switch to changesets, which was designed for that shape.

**Credentials and idempotency.** Auth is entirely environment-variable driven (`GITHUB_TOKEN`, `NPM_TOKEN`, etc.); the `verifyConditions` step fails fast if a token is missing or lacks scope, which is good. But a partial failure mid-publish (tag created, npm publish failed) can leave the release half-done and needs manual reconciliation. Always validate config with `npx semantic-release --dry-run` before wiring it into a protected branch.

## When to Use / When Not

**Use when:**
- You publish a single package (npm or otherwise) and want zero-touch, convention-driven releases from CI.
- Your team already writes — or is willing to enforce — Conventional Commits.
- You want the release history and changelog to be a deterministic function of git, with no manual version bumps.

**Avoid when:**
- You have a monorepo needing independent per-package versioning — reach for changesets instead.
- Your team won't commit to a commit-message convention; the tool degrades to silent no-ops.
- You want a human to review/approve version numbers and release notes before they ship — semantic-release is deliberately unattended.
- You need to run releases interactively from a laptop rather than CI.

## Alternatives

- changesets/changesets — use instead when you have a monorepo and want intent-based, human-authored changesets rather than commit-message inference.
- googleapis/release-please — use instead when you want a "release PR" that batches changes for human approval before publishing, still from Conventional Commits.
- release-it/release-it — use instead when you want an interactive, locally-run release tool with prompts rather than fully automated CI.
- goreleaser/goreleaser — use instead for Go projects that need cross-compiled binaries and multi-platform artifacts.
- conventional-changelog/standard-version — the older changelog-only tool; effectively deprecated, but relevant when you only want version+changelog without publishing.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2014 | Created by Stephan Bönnemann; commit-driven release automation[^2]. |
| 15 | 2019 | Plugin-based lifecycle matured; org split into per-plugin repos. |
| 20 | 2023 | ESM-only migration; raised Node baseline[^4]. |
| 24 | 2024–2025 | Current major line; continued Node-floor and plugin updates. |

## References

[^1]: semantic-release README — "removes the immediate connection between human emotions and version numbers." https://github.com/semantic-release/semantic-release
[^2]: semantic-release org and history; project created 2014. https://github.com/semantic-release
[^3]: Release steps / lifecycle documentation. https://semantic-release.org/extending/plugins-list/
[^4]: semantic-release v20 ESM migration and Node support policy. https://semantic-release.org/support/node-version/

## Tags

javascript, release-automation, semver, ci-cd, changelog, npm-publishing, conventional-commits, devops, versioning, nodejs
