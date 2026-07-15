# orhun/git-cliff

> A configurable changelog generator that turns conventional-commit history into formatted release notes via a template engine.

[GitHub repo](https://github.com/orhun/git-cliff) ·
[Official website](https://git-cliff.org) ·
[License: Apache-2.0 OR MIT](https://github.com/orhun/git-cliff/blob/main/LICENSE-APACHE)

## Overview

git-cliff is a Rust CLI that reads a Git repository's commit history and renders a CHANGELOG from it. It parses commit messages — typically following the [Conventional Commits](https://www.conventionalcommits.org/) spec, though arbitrary regex parsers are supported — groups them by type (features, fixes, etc.), and feeds the result through a [Tera](https://keats.github.io/tera/) template to produce Markdown (or any text format). It was started by Orhun Parmaksız in 2021 and has become one of the more widely adopted changelog tools in the Rust ecosystem and beyond[^1].

The project's defining characteristic is that almost nothing is hardcoded. Commit grouping, message preprocessing, tag filtering, link expansion, and output layout are all driven by a single `cliff.toml` file. This is the tradeoff at the center of the tool: it will match nearly any changelog convention you already have, but doing so means learning both the config schema and Tera's Jinja2-like template syntax. For teams that already practice conventional commits and want the default `keepachangelog` output, it is close to zero-config; for everyone else, the customization surface is large.

Since 2.0 (2024) git-cliff also enriches changelogs with data pulled from remote forges — GitHub, GitLab, Gitea, and Bitbucket — resolving pull-request numbers, contributor handles, and first-time-contributor status[^2]. This shifted it from a pure offline history formatter toward a release-notes tool that depends on network access and API tokens for its richest output.

## Getting Started

```bash
# Rust toolchain
cargo install git-cliff
# or: brew install git-cliff / pacman -S git-cliff / npm install -g git-cliff
```

```bash
# Scaffold a default config, then generate the full changelog
git cliff --init
git cliff -o CHANGELOG.md

# Only commits since the latest tag, rendered as the "unreleased" section
git cliff --unreleased --tag v1.2.0

# Compute the next semver version from commit types and write it
git cliff --bumped-version
```

```toml
# cliff.toml (excerpt) — commit_parsers map messages to changelog groups
[git]
conventional_commits = true
filter_unconventional = true
commit_parsers = [
  { message = "^feat", group = "Features" },
  { message = "^fix", group = "Bug Fixes" },
  { message = "^chore\\(release\\)", skip = true },
]
```

## Architecture / How It Works

git-cliff is split into a thin `git-cliff` binary and a `git-cliff-core` library crate, the latter published on crates.io and usable programmatically[^3]. The pipeline is: read commits via `gix`/`git2`, parse each message against the configured `commit_parsers`, apply `commit_preprocessors` (regex substitutions, e.g. linkifying issue numbers), assemble commits into `Release` structs keyed by tag, optionally enrich releases with remote API data, then render everything through a Tera template.

Configuration lives in `cliff.toml` (or a `[tool.git-cliff]` table in other files). The three moving parts a user actually tunes are: **commit_parsers** (regex → group / skip / scope decisions), the **body template** (Tera, iterating over `commits` grouped by `group`), and **tag_pattern / topo-order** settings that decide what counts as a release boundary. Several built-in configs ship with the binary (`--init keepachangelog`, `github`, etc.) so common conventions don't require writing a template from scratch.

Conventional-commit parsing is handled through the `git-conventional` parser; commits that don't parse can be dropped (`filter_unconventional`) or captured by a fallback parser. Version bumping (`--bump`) inspects commit types to decide major/minor/patch increments, which is what lets git-cliff double as a release-version calculator, not just a formatter.

The remote integration is optional and lazy: it only activates when the template references remote fields or when `--github-repo`-style flags are set, and it requires an auth token to avoid low anonymous rate limits.

## Production Notes

- **Garbage in, garbage out.** The quality of the output is a direct function of commit-message discipline. Repositories that don't follow a consistent convention need hand-written `commit_parsers`, and getting those regexes right is the bulk of real-world setup time. Non-conforming commits are silently dropped when `filter_unconventional = true`.
- **Tera template errors surface at runtime.** A typo in the body template produces a Tera error at generation time, not a config-load error, and the messages can be terse. Keep templates under version control and test on a throwaway range before wiring into release automation.
- **Remote enrichment needs a token and costs latency.** Resolving PR/contributor metadata hits the forge API; without `GITHUB_TOKEN` (or the GitLab/Gitea/Bitbucket equivalent) you hit anonymous rate limits quickly on large histories. This step, not commit parsing, is the slow part of a run.
- **Monorepos are supported but manual.** `--include-path` / `--repository` scope generation to a subdirectory or multiple repos, but you configure the boundaries yourself; there is no automatic package graph. Related tooling (release-plz) builds workspace awareness on top of git-cliff-core.
- **Shallow clones break tag detection.** CI checkouts default to shallow history; `git-cliff` needs full history and tags to compute releases, so fetch depth and `fetch-tags` must be set explicitly in the pipeline.
- **Determinism vs. remote data.** Output that embeds contributor/PR data is only reproducible while the remote state and token are available; offline runs of the same config will render a reduced changelog.

## When to Use / When Not

**Use when:**
- Your project already uses (or is willing to adopt) conventional commits and you want release notes generated from that history.
- You need a specific changelog format and want one config/template to enforce it across repos.
- You want changelog generation and next-version calculation from the same tool, runnable in CI, pre-commit, or a GitHub Action.
- You're in the Rust ecosystem, though the tool is language-agnostic.

**Avoid when:**
- Your commit history is unstructured and you're unwilling to write parsers — the output will be thin or require heavy regex work.
- You want changelogs authored from pull-request titles/labels rather than commit messages; forge-native tools fit that model better.
- You need an integrated end-to-end release manager (tagging, publishing, PR creation) out of the box rather than the changelog/version-bump piece.

## Alternatives

- oknozor/cocogitto — conventional-commit toolkit with changelog, bump, and commit linting; more opinionated, less template-flexible.
- MarcoIeni/release-plz — full Rust release automation that uses git-cliff-core internally; choose it when you want PR-driven releases, not just a changelog.
- googleapis/release-please — PR-based release automation driven by conventional commits; use when your workflow is GitHub-PR-centric across many languages.
- conventional-changelog/conventional-changelog — the original Node.js ecosystem standard; use in JS-heavy projects already on that toolchain.
- saschagrunert/git-journal — commit-message and changelog framework with its own message format; use when you want an enforced template rather than conventional commits.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2021 | Initial release: Tera templating over conventional commits[^1]. |
| 1.0.0 | 2022 | Config schema stabilized; built-in preset configs. |
| 2.0.0 | 2024 | Remote integration (GitHub/GitLab/Gitea/Bitbucket), contributor metadata[^2]. |
| 2.x | 2024–2026 | Ongoing: version bumping, monorepo flags, config refinements[^4]. |

## References

[^1]: git-cliff repository and documentation. https://github.com/orhun/git-cliff — https://git-cliff.org/docs
[^2]: git-cliff remote/GitHub integration docs. https://git-cliff.org/docs/integration/github
[^3]: `git-cliff-core` crate on docs.rs. https://docs.rs/git-cliff-core/
[^4]: git-cliff releases. https://github.com/orhun/git-cliff/releases

## Tags

rust, changelog, changelog-generator, conventional-commits, cli, release-automation, git, semver, tera-templates, developer-tools
