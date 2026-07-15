# pre-commit/pre-commit

> A language-agnostic framework for managing git pre-commit hooks, with each hook running in its own isolated, auto-provisioned environment.

[GitHub repo](https://github.com/pre-commit/pre-commit) ·
[Official website](https://pre-commit.com) ·
[License: MIT](https://github.com/pre-commit/pre-commit/blob/main/LICENSE)

## Overview

pre-commit is a git hook manager written in Python but deliberately not tied to Python projects. It reads a `.pre-commit-config.yaml` at the repo root, and for each hook it clones the hook's source repo, builds an isolated environment in the language that hook declares (Python venv, Node, Ruby, Rust, Go, Docker, and others), and runs it against your staged files on `git commit`. It was created by Anthony Sottile and collaborators, originating out of tooling work at Yelp, and has become the de facto standard for pre-commit automation across ecosystems[^1].

Its defining decision is the isolation model: a hook you reference (say, `black` or `ruff` or `eslint`) is not something you install into your project's dependencies. pre-commit provisions it separately, keyed by the pinned `rev` in your config, and caches it under `~/.cache/pre-commit`. This is what lets a single config coordinate linters written in five different languages without polluting the project, and it is also the source of most of the tool's friction — the first run is slow, the cache grows, and each hook repo must ship a `.pre-commit-hooks.yaml` manifest describing how to build and invoke it.

The central tension is reproducibility versus speed and network dependence. Pinning every hook to a git `rev` gives byte-stable, auditable tooling across a whole team and CI; it also means the first commit on a fresh checkout clones repos and compiles environments, and an offline machine with a cold cache cannot run hooks at all.

## Getting Started

```bash
pip install pre-commit          # or: brew install pre-commit / pipx install pre-commit
pre-commit install              # writes .git/hooks/pre-commit
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff          # lint
      - id: ruff-format   # format
```

```bash
pre-commit run --all-files      # run against the whole repo, not just staged files
pre-commit autoupdate           # bump each repo's rev to its latest tag
```

## Architecture / How It Works

A hook is defined by a **hook repository** that ships a `.pre-commit-hooks.yaml` manifest at its root. Each entry declares an `id`, an `entry` command, a `language`, and file-matching metadata (`types`, `files`, `exclude`). When you reference that repo with a `rev` in your config, pre-commit clones it at that revision into its store and builds an environment according to the declared `language`.

The supported languages are the real substance of the project: `python` (venv + pip), `node` (a bundled nodeenv), `ruby` (rbenv-style ruby-build), `rust` (cargo), `golang`, `docker` / `docker_image`, `dotnet`, `conda`, `coursier`, plus escape hatches — `system` (run an already-installed binary), `script`, `pygrep` (regex checks with no environment at all), and `fail`. Each language plugin knows how to create an isolated install and how to invoke the entry point inside it.

At commit time the flow is: stash unstaged changes, compute the set of files git is about to commit, filter them per hook by type and pattern, then invoke each hook's entry with the matching filenames as arguments. Hooks that mutate files (formatters) cause the commit to abort if they change anything, so you re-stage and commit again. Unstaged changes are restored from the stash afterward — this stash/restore dance is what makes pre-commit correctly lint only what is actually being committed, even with partially-staged files.

pre-commit is not limited to the `pre-commit` git hook. `pre-commit install --hook-type pre-push` (and `commit-msg`, `prepare-commit-msg`, `post-checkout`, `post-merge`, and others) wires the same config into other stages, gated by each hook's `stages` field. There are also **meta hooks** (`check-hooks-apply`, `check-useless-excludes`, `identity`) that lint the config itself.

## Production Notes

**First-run cost and the cache.** The initial `pre-commit run` on a cold `~/.cache/pre-commit` clones every hook repo and compiles every environment; for a config with Node and Ruby and Rust hooks this can take minutes. CI must cache that directory (keyed on a hash of `.pre-commit-config.yaml`) or pay the cost on every job. The cache also only grows — `pre-commit gc` prunes environments for revs no longer referenced, and `pre-commit clean` nukes it entirely.

**CI usage.** In CI you almost always want `pre-commit run --all-files`, because the git-hook path only sees a diff. The maintainers also run **pre-commit.ci**, a hosted service that runs the hooks on PRs, auto-fixes and pushes formatting changes, and periodically opens `autoupdate` PRs. It is a separate product from the CLI but tightly integrated via a `ci:` block in the config.

**The stash footgun.** Because pre-commit stashes unstaged changes during a run, a hook or an interrupted run (or an editor writing to a file mid-commit) can occasionally leave changes in the stash if something crashes hard. It is robust in practice, but merge-conflict states and manual `git` surgery mid-commit are where surprises live.

**Pinning and drift.** Every `rev` is a hard pin, which is the point — but it means your linters do not update until someone runs `autoupdate`. Teams that forget end up years behind. Conversely, `autoupdate` moves to the latest tag with no regard for breaking changes in the linter, so an unattended bump can suddenly fail CI. Use `language: system` hooks when you deliberately want the project's own pinned dependency (e.g. the `ruff` already in your lockfile) instead of a second isolated copy.

**Version-4 stage rename.** pre-commit 3.2 introduced git-native stage names (`pre-commit`, `pre-push`) and deprecated the old short names (`commit`, `push`); 4.0 removed the deprecated names outright. Configs and third-party hook manifests still using `stages: [commit]` break on 4.x and must migrate to `stages: [pre-commit]`[^2].

**Offline and air-gapped.** With a warm cache pre-commit runs offline, but provisioning a new hook needs network access to clone the repo and pull language packages. Fully air-gapped setups vendor hook repos locally and point `repo:` at a local path or use `repo: local` hooks.

## When to Use / When Not

**Use when:**
- Your repo mixes languages and you want one config to coordinate all the linters/formatters.
- You want reproducible, pinned tool versions shared across a team and CI.
- You want formatting and lint gates enforced before code ever reaches review.
- You want hooks that are trivial to add from the large existing ecosystem of `.pre-commit-hooks.yaml` repos.

**Avoid when:**
- Your project is pure JavaScript/TypeScript and you already live in Node — husky + lint-staged is a lighter, npm-native fit.
- You need maximum hook speed with parallel execution as a first-class feature — lefthook is built around that.
- Your environment is strictly offline with no cache-priming step — the clone-and-provision model fights you.
- You only need one linter you already run in CI; a git hook script may be simpler than the framework.

## Alternatives

- evilmartians/lefthook — single Go binary, parallel execution, YAML config; faster and dependency-free. Use instead when hook speed and zero runtime deps matter more than the cross-language hook ecosystem.
- typicode/husky — the standard git-hooks tool in the Node world. Use instead when your project is JS/TS-native and you want npm-managed hooks.
- okonet/lint-staged — runs commands against staged files; not a hook manager itself, usually paired with husky. Use instead (with husky) for a minimal JS staged-file linting setup.
- sds/overcommit — Ruby-based hook manager with a similar config model. Use instead in a Ruby-centric shop already invested in the toolchain.
- Plain `.git/hooks` scripts — no framework. Use instead when you have one simple check and want zero dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014 | Initial development; repo created March 2014[^1]. |
| 1.0.0 | 2017-10 | First stable release. |
| 2.0.0 | 2020-01 | Dropped Python 2 support; Python 3 only[^3]. |
| 3.0.0 | 2023-02 | Dropped older Python versions; internal cleanups[^4]. |
| 3.2.0 | 2024 | Git-native stage names (`pre-commit`, `pre-push`); old names deprecated. |
| 4.0.0 | 2024-10 | Removed deprecated short stage names; `stages: [commit]` no longer valid[^2]. |

## References

[^1]: pre-commit — official site and documentation. https://pre-commit.com/
[^2]: pre-commit 4.0 changelog — deprecated stage-name removal. https://github.com/pre-commit/pre-commit/blob/main/CHANGELOG.md
[^3]: pre-commit 2.0 release notes — Python 2 removal. https://github.com/pre-commit/pre-commit/releases
[^4]: pre-commit 3.0 release notes. https://github.com/pre-commit/pre-commit/releases

## Tags

python, git, git-hooks, linting, formatting, developer-tooling, ci, code-quality, pre-commit, static-analysis
