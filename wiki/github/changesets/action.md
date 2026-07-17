# changesets/action

> GitHub Action that turns pending changeset files into an auto-maintained "Version Packages" PR — merge the PR to version, changelog, and publish your packages to npm.

[GitHub repo](https://github.com/changesets/action) ·
[Official website](https://changesets.dev) ·
[License: MIT](https://github.com/changesets/action/blob/main/LICENSE)

## Overview

`changesets/action` is the CI half of the [Changesets](https://github.com/changesets/changesets) release workflow for JavaScript/TypeScript repos, especially multi-package monorepos. Contributors commit small markdown "changeset" files declaring intent ("bump `@scope/pkg` minor: added X"); this action, running on every push to the base branch, collects them into a single pull request with all version bumps and changelog entries applied. Merging that PR triggers the same action to publish to npm, push git tags, and create GitHub releases[^1]. The repo dates to August 2019 and remains actively maintained (pushed this week); at ~1,000 stars the count badly understates adoption — as a CI dependency it sits in the release pipelines of much of the JS monorepo ecosystem (pnpm, emotion, XState, and most Changesets-org-adjacent projects), where nobody stars the action itself.

The design's defining tension: it drives releases *through GitHub's PR model using GitHub's own token*, and GitHub deliberately prevents `GITHUB_TOKEN`-created commits and PRs from triggering other workflows. So the Version Packages PR the action opens gets no CI runs by default — the single most common operational surprise, with a well-worn workaround (PAT or GitHub App token) that shifts secret-management burden onto you.

Versioning is currently split: the `v1` line (stable, `maintenance/v1` branch) targets Changesets v2, while `main` is the in-progress `v2` targeting Changesets v3, published only as `v2.0.0-next.*` prereleases as of mid-2026[^2]. The v2 rewrite renames all inputs to kebab-case (`publish` → `publish-script`, `version` → `version-script`), so examples on the internet are split between two incompatible input schemas.

## Getting Started

Requires an existing Changesets setup (`npx changeset init` creates `.changeset/`). Then add `.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    branches: [main]

concurrency: ${{ github.workflow }}-${{ github.ref }}

permissions:
  contents: write
  pull-requests: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: 26
      - run: npm ci
      - name: Create Release PR or Publish
        id: changesets
        uses: changesets/action@v1
        with:
          publish: npx changeset publish   # v2 prerelease: `publish-script:`
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

The `NPM_TOKEN` must be able to publish without 2FA-on-publish (an npm automation/granular token)[^1].

## Architecture / How It Works

The action is a TypeScript program with one branching decision, evaluated on every push to the base branch:

1. **Changesets pending** (`.changeset/*.md` files exist): run `changeset version` (or your `version` script), which consumes the changeset files, bumps `package.json` versions across the monorepo, and rewrites changelogs. The result is committed to a `changeset-release/<branch>` branch, **force-pushed**, and an open "Version Packages" PR is created or updated. The PR body aggregates the pending changelog entries.
2. **No changesets pending** and a `publish` script is configured: assume a Version Packages PR was just merged, run the publish script (conventionally `changeset publish`, which skips already-published versions), then push git tags and create GitHub releases for each published package, and expose `published` / `published-packages` outputs for downstream steps (notifications, deploys)[^1].

The action itself contains no versioning logic — semver math, changelog generation, and inter-package dependency bumping all live in the `changesets` CLI it shells out to. This keeps the action thin but couples its major versions to Changesets major versions (v1↔Changesets v2, v2↔Changesets v3)[^2].

Notable knobs: `commit-mode: github-api` routes commits/tags through the GitHub API instead of git CLI, producing GPG-signed commits attributed to the token owner (needed for orgs enforcing signed commits); `pr-draft` controls draft-PR behavior; the action writes an `~/.npmrc` with `NPM_TOKEN` unless one already exists. Two sub-actions live in the same repo: `pr-status` (changeset status reporting in PRs) and `pr-comment`[^1].

## Production Notes

**The Version Packages PR gets no CI.** With the default `GITHUB_TOKEN`, GitHub suppresses workflow triggers on the PR the action creates[^3]. Teams routinely merge release PRs that were never tested. Fixes: pass a PAT or GitHub App installation token as `github-token`, or trigger CI on `push` to `changeset-release/**`. Budget for token rotation either way.

**Permissions.** Under GitHub's restricted default token permissions you must grant `contents: write` and `pull-requests: write`, or the action fails with an unhelpful 403. `commit-mode: github-api` plus GitHub releases also needs the token to create releases.

**Race conditions.** Two pushes to `main` in quick succession can run the publish path concurrently. The README's `concurrency: ${{ github.workflow }}-${{ github.ref }}` line is not decoration — omit it and you will eventually see duplicate-publish errors. Relatedly, a changeset-free commit landing after a publish re-enters the publish branch; `changeset publish` tolerates this (skips existing versions), but custom publish scripts must be idempotent[^1].

**Force-push semantics.** The `changeset-release/*` branch is force-pushed on every base-branch push. Manual edits to the release PR branch (e.g., hand-tuning a changelog) are silently destroyed the next time anyone merges to `main`. Edit the changeset files on `main` instead.

**Changelog generators need the token in env.** `@changesets/changelog-github` (the common choice, links PRs/authors) reads `GITHUB_TOKEN` from the environment of the *version* step; forgetting it fails the version run.

**Maintenance cadence.** Release history shows long quiet stretches (one patch release in all of 2024's mid-year, v1.4.7→v1.4.9) followed by a marked 2026 revival: v1.6.0 through v1.9.0 shipped January–June 2026 alongside the v2 prerelease push[^2]. 120 open issues reflect a small-maintainer-team project carrying a large user base, not abandonment.

**Pinning.** `@v1`/`@v2` are floating tags. Security-sensitive repos should pin to a commit SHA — this action handles your npm credentials.

## When to Use / When Not

**Use when:**
- You run a JS/TS monorepo publishing multiple interdependent npm packages and already use (or want) Changesets' contributor-authored release notes.
- You want a human review gate (the Version Packages PR) between "code merged" and "version published".
- You want batched releases: many merged PRs → one version PR → one publish.

**Avoid when:**
- You want zero-touch per-commit releases from commit messages — semantic-release's model, no PR gate, no changeset files.
- Your packages are not npm/JS: Changesets is JS-ecosystem-specific; release-please covers many languages.
- You need immediate publishes; the PR-gate model adds a human merge step by design.
- Your team won't reliably write changeset files — the whole pipeline silently does nothing without them (the `pr-status` sub-action or the changeset-bot mitigates this).

## Alternatives

- googleapis/release-please — same release-PR model, conventional-commit-driven and multi-language; use it outside the JS/Changesets ecosystem or when commit messages should drive versioning.
- semantic-release/semantic-release — fully automated publish on every qualifying commit, no review gate; use when you want releases with zero human steps.
- intuit/auto — label-driven releases from PR labels; use when PR metadata, not files or commit syntax, should decide bumps.
- lerna/lerna — older monorepo versioning/publishing (`lerna version`/`lerna publish`); use for legacy Lerna repos not worth migrating.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-08 | Repo created; pre-1.0 usage via `@master`. |
| v1.0.0 | 2021-11-25 | First tagged stable release[^2]. |
| v1.4.0 | 2022-11-02 | v1 line matures; slow-cadence maintenance era begins. |
| v1.4.9 | 2024-10-14 | One of only three releases across 2024 — quietest period. |
| v1.5.0 | 2025-05-05 | Maintenance resumes; rapid 1.5.x patch series in May 2025. |
| v1.6.0–v1.9.0 | 2026-01 → 2026-06 | Active 2026 feature cadence on the v1 line[^2]. |
| v2.0.0-next.0 | 2026-06-23 | Changesets v3 compatibility; kebab-case input renames; v1 moves to `maintenance/v1`[^2]. |

## References

[^1]: changesets/action README (main branch). https://github.com/changesets/action#readme
[^2]: changesets/action releases (tag dates). https://github.com/changesets/action/releases
[^3]: GitHub Docs, "Triggering a workflow from a workflow" — events from `GITHUB_TOKEN` do not create new workflow runs. https://docs.github.com/en/actions/using-workflows/triggering-a-workflow#triggering-a-workflow-from-a-workflow

## Tags

github-actions, typescript, release-automation, versioning, changelog, monorepo, npm-publishing, ci-cd, semver, javascript
