# changesets/changesets

> A monorepo-first workflow for declaring version bumps and changelog entries as files, then batching them into releases.

[GitHub repo](https://github.com/changesets/changesets) ·
[Official website](https://changesets.dev) ·
[License: MIT](https://github.com/changesets/changesets/blob/main/LICENSE)

## Overview

Changesets solves a narrow, real problem: in a multi-package repository, deciding *what* to release, at *what* semver bump, with *what* changelog text, is normally a manual and error-prone step done at release time. Changesets moves that decision to the moment a change is made. A contributor writes a small markdown file — a "changeset" — that names the affected packages, the bump type for each, and a human summary. At release time a single command consumes all accumulated changesets, flattens them into one version bump per package, rewrites `package.json` versions, updates `CHANGELOG.md` files, and (optionally) publishes to npm[^1].

The project originated inside Atlassian and was open-sourced and rearchitected with sponsorship from Thinkmill, the consultancy behind KeystoneJS[^2]. It is written in TypeScript and distributed as a set of scoped packages under `@changesets/*`, with `@changesets/cli` as the entry point. Its defining tension is scope: changesets deliberately does *not* infer releases from commit messages (the semantic-release model). It requires an explicit, reviewable artifact per change. This is more friction than commit-convention tooling, and the tradeoff is intentional — the changeset file is a durable, editable record that survives squash-merges and lets a human write a changelog entry that reads for users rather than for machines.

As of 2026 the tool is widely adopted across the JavaScript ecosystem — pnpm, Astro, SvelteKit, Biome, Apollo Client, Chakra UI, and Remix among them[^3] — and remains actively maintained, with the `main` branch tracking an in-progress v3 while v2 continues on a `maintenance/v2` branch[^4]. The relatively high open-issue count reflects a broad feature-request backlog against a small maintainer team, not abandonment.

## Getting Started

```bash
# Install the CLI into a workspace root (npm/yarn/pnpm workspaces all supported)
npm install --save-dev @changesets/cli
npx changeset init          # creates .changeset/ with config.json + README
```

```bash
# Author a changeset (interactive: pick packages, bump types, write summary)
npx changeset
```

This writes a file like `.changeset/tame-lions-drum.md`:

```markdown
---
"@myscope/button": minor
"@myscope/theme": patch
---

Add `variant="ghost"` to Button and adjust default focus ring in theme.
```

```bash
# At release time — usually run by CI, not a human:
npx changeset version   # consume all changesets → bump versions + changelogs
npx changeset publish    # publish changed packages to npm, create git tags
```

## Architecture / How It Works

The workflow is a two-phase pipeline separated in time. **Authoring** (`changeset add`, the default command) writes markdown-with-frontmatter files into `.changeset/`. Each file is independent; several PRs can each add their own without conflict, which is the property that makes changesets survive parallel development and squash-merges. **Consumption** (`changeset version`) reads every changeset in the directory, groups bumps per package, takes the highest bump for each, applies them to `package.json`, and deletes the consumed changeset files in the same commit.

The non-obvious internal work happens between those two steps: **internal dependency graph resolution**. When package A depends on package B and B receives a bump, changesets bumps A too (a patch by default) and updates A's dependency range on B. `updateInternalDependencies` and the dependency's range style (`^`, `~`, `workspace:*`) govern exactly how ranges are rewritten[^5]. This is the part that hand-rolled release scripts get wrong, and it is the core value of the tool over `npm version` in a monorepo.

Two grouping mechanisms sit on top of the graph. **Fixed** packages are always released together at the same version, even if only one changed — the pattern React and Babel use. **Linked** packages share a version line but only bump the ones that actually changed[^5]. Both are configured as glob arrays in `.changeset/config.json`.

Changelog generation is pluggable. The default writer emits a terse list; `@changesets/changelog-github` produces entries with PR/commit/author links but requires a `GITHUB_TOKEN` at version time[^6]. `changeset publish` wraps `npm publish` per changed package, respects each package's `publishConfig` and `access` field, and creates git tags — it does not push them, leaving that to CI.

Prerelease and snapshot modes reuse the same pipeline. `changeset pre enter <tag>` puts the repo into a prerelease state so `version` produces `1.2.0-beta.0` style bumps until `changeset pre exit`. `changeset version --snapshot` produces throwaway timestamp/hash versions for per-commit canary publishes[^7].

## Production Notes

- **The `access` footgun.** Scoped packages default to `restricted` on npm. If `publishConfig.access` (or the changeset `access` config) is not set to `public`, `changeset publish` fails on first publish with a permissions error. This is the single most common first-release failure.
- **CI is the intended runner, not humans.** The canonical setup is `changesets/action`, a GitHub Action that opens and maintains a "Version Packages" PR: it runs `changeset version` on every push to the base branch and, when that PR merges, runs `changeset publish`[^8]. Running `version`/`publish` from a laptop works but drifts from the reviewed-PR model the tool is built around.
- **Enforcing a changeset per PR** is not automatic. Teams add the changeset bot[^9] to comment on PRs missing one, or run `changeset status --since=main` in CI to fail the build. Neither is installed by default.
- **Empty changesets are a real workflow.** `changeset add --empty` records "this PR intentionally releases nothing," which keeps the bot quiet without forcing a version bump. New users often don't know this exists and bump packages spuriously.
- **Non-npm and private packages.** Changesets is npm-package-shaped. Versioning applications, Docker images, or non-JS artifacts is possible but awkward and documented as a workaround, not a first-class path[^10]. `privatePackages` config controls whether private packages get versioned and/or tagged.
- **Snapshot version collisions.** Snapshot releases publish under a dist-tag; forgetting `--tag` on `publish` can move `latest` to a canary build. Always pair `changeset publish --tag <name>` with snapshot versioning.
- **Monorepo tool interplay.** Works with pnpm/yarn/npm workspaces, but pnpm `workspace:` protocol ranges and Yarn Berry constraints each have edge cases in how ranges get rewritten; verify the first release's diff by hand.

## When to Use / When Not

**Use when:**
- You maintain a multi-package repo (npm workspaces / pnpm / yarn) and need per-package semver with correct internal-dependency bumping.
- You want changelog entries written by humans for users, reviewable in the same PR as the code.
- You want release automation that keeps the actual publish behind a merge of a visible "Version Packages" PR.

**Avoid when:**
- You publish a single package and are happy inferring releases from Conventional Commits — semantic-release is less ceremony.
- You want fully commit-message-driven releases with zero extra files per PR.
- Your artifacts aren't npm packages (containers, apps, multi-language monorepos) — the fit degrades and other tools model that natively.

## Alternatives

- semantic-release/semantic-release — use instead when you have a single package and want releases derived automatically from Conventional Commit messages with no per-change file.
- googleapis/release-please — use instead when you want PR-based, Conventional-Commits-driven releases across multiple languages, not just npm.
- lerna/lerna — use instead when you want an all-in-one monorepo task runner whose versioning is part of a larger toolchain (now Nx-maintained).
- nrwl/nx — use instead when you already run Nx and want release/versioning integrated with its task graph via `nx release`.
- microsoft/rushstack — use instead when you run Rush; its `rush change` files are the same idea, coupled to Rush's stricter monorepo policy model.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-03 | Repository created; open-sourced from Atlassian's internal tooling[^2]. |
| 1.x | 2019 | First public `@changesets/cli`; core author/consume workflow. |
| 2.0 | 2020 | Rearchitecture sponsored by Thinkmill; linked/fixed packages, config-driven changelog writers[^2]. |
| — | 2020–2021 | `changesets/action` and the changeset bot establish the CI-driven release pattern[^8]. |
| 3.x (dev) | 2026 | v3 in progress on `main`; v2 continues on `maintenance/v2`[^4]. |

## References

[^1]: changesets README — "Intro / How do we do that?". https://github.com/changesets/changesets#readme
[^2]: changesets README — Thanks/Inspiration (Atlassian original, Thinkmill-sponsored v2 rearchitecture). https://github.com/changesets/changesets#thanksinspiration
[^3]: changesets README — "Cool Projects already using Changesets". https://github.com/changesets/changesets#readme
[^4]: changesets README top note — v3 development on `main`, v2 on `maintenance/v2`. https://github.com/changesets/changesets/tree/maintenance/v2
[^5]: Fixed and linked packages docs. https://github.com/changesets/changesets/blob/main/docs/linked-packages.md
[^6]: Modifying changelog format. https://github.com/changesets/changesets/blob/main/docs/modifying-changelog-format.md
[^7]: Snapshot releases and prereleases docs. https://github.com/changesets/changesets/blob/main/docs/snapshot-releases.md
[^8]: changesets/action — GitHub Action for versioning/publishing PRs. https://github.com/changesets/action
[^9]: changeset-bot — GitHub app that checks PRs for changesets. https://github.com/apps/changeset-bot
[^10]: Versioning applications and other non-npm packages. https://github.com/changesets/changesets/blob/main/docs/versioning-apps.md

## Tags

typescript, javascript, monorepo, versioning, changelog, semver, npm-publishing, release-automation, cli, developer-tools, ci-cd
