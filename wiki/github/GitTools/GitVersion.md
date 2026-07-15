# GitTools/GitVersion

> Derives a Semantic Version from your git history and branch layout, so you never hand-edit a version number in CI again.

[GitHub repo](https://github.com/GitTools/GitVersion) ·
[Official website](https://gitversion.net/docs/) ·
[License: MIT](https://github.com/GitTools/GitVersion/blob/main/LICENSE)

## Overview

GitVersion is a .NET command-line tool (and MSBuild task) that reads a repository's
commit graph, tags, and current branch, and computes a [SemVer](https://semver.org)
version string for the commit being built. The premise is that the version number is
already implied by git — the last release tag, the number of commits since it, and the
branch you are on — so a tool should derive it deterministically rather than have humans
bump numbers by hand. It began life around 2013 as "GitFlowVersion" and was renamed once
its scope grew beyond the GitFlow branching model[^1].

The defining tension is convention vs. configuration. GitVersion ships opinionated defaults
tied to well-known branching strategies (GitFlow, GitHubFlow, trunk/mainline), and when
your repository matches one of those exactly, it is close to zero-config. The moment your
branch names, release process, or tag conventions diverge, you are writing a
`GitVersion.yml` and learning the tool's model of increments, version-source commits, and
branch regexes. It is powerful for teams that want versions to fall out of git automatically,
and a source of frustration for teams whose git history does not resemble the assumed shape.

GitVersion is widely embedded in .NET CI pipelines but is language-agnostic in output — it
just prints a version, which any build (npm, Docker, MSBuild, shell) can consume. It is
distributed as a dotnet global tool, an MSBuild NuGet package, a Chocolatey portable exe, a
Homebrew formula, a Docker image, an Azure DevOps task, and a GitHub Action.

## Getting Started

```bash
# Install as a .NET global tool (requires the .NET SDK)
dotnet tool install --global GitVersion.Tool

# Run at the root of a git repo; prints a JSON version object
dotnet-gitversion

# Grab a single field for a build script
dotnet-gitversion /showvariable SemVer      # e.g. 1.4.0-beta.5
```

```yaml
# GitVersion.yml — minimal config selecting a workflow preset
workflow: GitHubFlow/v1
next-version: 1.0.0
```

For MSBuild projects, add the `GitVersion.MsBuild` package instead and the version is
injected into the assembly at build time with no CLI call.

## Architecture / How It Works

At the core, GitVersion walks the commit graph using LibGit2Sharp (managed bindings over
libgit2). It locates a "version source" — normally the most recent reachable tag that looks
like a version — then counts commits from that point to `HEAD` to compute the pre-release
number and build metadata. The branch you are on selects a configuration block (via regex
match on the branch name) that decides how the base version is incremented (major/minor/patch),
what pre-release label to use (`beta`, `alpha`, `pr`, etc.), and whether the branch produces
stable or pre-release versions.

Configuration is layered: built-in defaults, a chosen workflow preset, and your
`GitVersion.yml` overrides merge together. Increment can also be driven per-commit through
commit-message bumpers (`+semver: minor`) for teams that want explicit control inside a
mainline model. Output is a set of variables (`SemVer`, `FullSemVer`, `NuGetVersion`,
`InformationalVersion`, `AssemblySemVer`, `Sha`, and more) emitted as JSON or exported as
environment/CI variables.

The single most important architectural fact is that GitVersion needs real git history to
work. It reasons over reachable commits and tags; if the clone is shallow or the tags were
not fetched, its answer is wrong or it errors out. Everything else about operating the tool
follows from that constraint.

## Production Notes

- **Shallow clones break it.** CI systems default to shallow clones for speed. In GitHub
  Actions you must set `fetch-depth: 0` on `actions/checkout`; in Azure Pipelines disable
  shallow fetch. A shallow clone is the number-one cause of "GitVersion produced the wrong
  number" or "repository has no tags" reports[^2].
- **Tags must be fetched.** Even a full-depth checkout can omit tags depending on the CI
  step; without the seed tag GitVersion counts from the root commit and inflates versions.
- **Detached HEAD.** Most CI checks out a detached commit rather than a named branch.
  GitVersion has heuristics to recover the branch, but ambiguous cases (a commit reachable
  from several branches) can pick a different config block than you expect. Pinning the
  branch or using the CI's source-branch variable avoids surprises.
- **Performance on large histories.** Walking a graph with tens of thousands of commits is
  measurably slower than a trivial `git describe`; the LibGit2Sharp path adds native-interop
  overhead. For very large monorepos this shows up as multi-second startup per invocation.
- **Config is a breaking-change surface.** GitVersion has changed its configuration schema
  across major versions — v5 reworked mainline handling, and v6 (2024) restructured the
  configuration model again, renaming and removing keys and replacing the old `mode` concept
  with workflow presets and deployment/versioning strategies[^3]. Upgrading across a major
  version almost always means rewriting `GitVersion.yml`; pin the tool version in CI and
  migrate deliberately.
- **Reproducibility.** Because the version depends on the full graph, two builds of the same
  commit are only identical if both see the same tags and branch. This is fine in disciplined
  pipelines and a footgun in ad-hoc local builds where a stale tag set gives a different answer
  than CI.

## When to Use / When Not

**Use when:**
- Your team follows a recognizable branching strategy (GitFlow, GitHubFlow, trunk-based) and
  wants versions derived automatically from it.
- You want rich outputs (NuGet, assembly, informational versions) from one source of truth.
- You are in the .NET ecosystem, where the MSBuild integration is close to invisible.

**Avoid when:**
- Your history is unusual, tags are inconsistent, or CI does shallow clones you cannot change.
- You prefer commit-message-driven releases (Conventional Commits) that also generate
  changelogs and publish — that is a different tool category.
- You want the simplest possible tag-plus-height scheme with no branch configuration.

## Alternatives

- adamralph/MinVer — minimal, tag-only versioning for .NET; no branch config, no YAML. Use when you want the simplest deterministic scheme and control releases purely with tags.
- dotnet/Nerdbank.GitVersioning — deterministic height-based versions from a `version.json`; use when you need cloud-build reproducibility and cross-language SDK support.
- semantic-release/semantic-release — Conventional-Commits-driven release automation for the JS ecosystem; use when you want the commit log to also drive changelog and publish steps.
- googleapis/release-please — PR-based release automation that opens a "release PR"; use when you want a human approval gate and generated changelogs over Node/multi-language repos.

## History

| Version | Date | Notes |
|---------|------|-------|
| —       | 2013 | Started as GitFlowVersion, later renamed to GitVersion[^1]. |
| 3.x     | 2016 | Widely adopted in .NET CI; GitFlow-centric configuration. |
| 5.0     | 2019 | Rewrite, added mainline mode, dotnet global tool packaging[^3]. |
| 6.0     | 2024 | Configuration model restructured; workflow presets, key renames/removals[^3]. |

## References

[^1]: GitVersion documentation, "Why GitVersion" and project history. https://gitversion.net/docs/learn/why
[^2]: GitVersion documentation, running in CI / requirement for non-shallow clones and tags. https://gitversion.net/docs/reference/requirements
[^3]: GitVersion documentation, configuration reference and version migration notes. https://gitversion.net/docs/reference/configuration

## Tags

csharp, dotnet, semver, versioning, git, ci-cd, build-tooling, cli, gitflow, release-automation
