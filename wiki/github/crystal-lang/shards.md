# crystal-lang/shards

> The dependency manager for the Crystal language — decentralized, git-based, bundled with every Crystal install.

[GitHub repo](https://github.com/crystal-lang/shards) ·
[Shards manual (crystal-lang.org)](https://crystal-lang.org/reference/man/shards/) ·
[License: Apache-2.0](https://github.com/crystal-lang/shards/blob/master/LICENSE)

## Overview

Shards is Crystal's official package manager, analogous to Bundler for Ruby or
Cargo for Rust — with one structural difference: **there is no central
registry**. Dependencies ("shards") are plain git repositories, referenced by
URL or by `github:`/`gitlab:`/`bitbucket:` shorthand in a `shard.yml` manifest,
with versions derived from `v`-prefixed semver-style git tags[^1]. Started by
Julien Portalier (ysbaddaden) in 2015, it was adopted under the crystal-lang
organization and ships inside official Crystal distributions, so effectively
every Crystal developer uses it. The 791 GitHub stars understate its reach for
exactly that reason — nobody installs it separately, it arrives with the
compiler. Commits into July 2026 confirm active maintenance in lockstep with
Crystal itself.

The registry-less design is the project's defining tradeoff. It eliminates the
central point of failure and the name-squatting economics of npm-style
registries; publishing is just pushing a git tag. In exchange, there is no
global namespace (two forks of the same shard are distinct dependencies), no
immutable artifact store (a re-pointed tag silently changes what installs),
and discovery is delegated to community sites like shardbox.org[^2].

## Getting Started

Shards comes with Crystal (Homebrew, apt, etc. — see crystal-lang.org/install).
In a project directory:

```bash
shards init      # generates a starter shard.yml
shards install   # resolves, installs into lib/, writes shard.lock
```

A typical `shard.yml`:

```yaml
name: my-app
version: 0.1.0

dependencies:
  kemal:
    github: kemalcr/kemal
    version: ~> 1.4

development_dependencies:
  ameba:
    github: crystal-ameba/ameba

license: MIT
```

Check both `shard.yml` and `shard.lock` into version control; subsequent
installs reproduce the locked versions[^1]. `shards update` re-resolves within
constraints; `shards build` compiles executables declared under `targets:`.

## Architecture / How It Works

Shards is a single static-friendly binary written in Crystal (building it from
source needs Crystal plus libyaml)[^1]. The moving parts:

- **Resolvers.** Each source type — git (the overwhelming default), local
  `path:` dependencies, and Mercurial/Fossil in newer versions — is a resolver
  that lists versions and materializes a chosen one. Git versions come from
  tags; branch and commit pins are also supported, in which case `shard.lock`
  records the exact commit SHA.
- **Version resolution.** Since 0.9.0 (2019), constraint solving uses a Crystal
  port of Molinillo, the backtracking dependency-resolution algorithm shared by
  Bundler and CocoaPods[^3]. Before that, resolution was naive and could fail
  on solvable graphs; the rewrite is why moderately deep Crystal dependency
  trees resolve reliably today.
- **Installation.** Resolved shards are copied into the project-local `lib/`
  directory (bare git mirrors are cached under `~/.cache/shards` to avoid
  re-cloning). The Crystal compiler's `require` lookup includes `lib/` — that
  directory is the entire integration contract between shards and the
  compiler. No runtime component, no loader: after install, the compiler just
  sees source files.
- **Postinstall hooks.** A shard may declare a `scripts: postinstall:` command
  that runs on install — used legitimately for building C extensions, but it
  is arbitrary shell execution (see Production Notes).

Identity is the resolver URL, not a package name: renaming a GitHub org or
switching between forks changes a dependency's identity even if the code and
`name:` field are the same.

## Production Notes

- **Tag mutability is your supply chain.** For tag-derived versions,
  `shard.lock` records the version, and a force-moved tag can change what a
  fresh install fetches. If that risk matters, pin critical dependencies to a
  `commit:` explicitly, or vendor `lib/` into your repo (it is plain source;
  committing it works fine).
- **Postinstall scripts run arbitrary code.** Audit new dependencies' `scripts`
  sections before first install; `--skip-postinstall` skips execution.
- **CI should use `shards install --production`** (or `--frozen`): it fails if
  `shard.lock` is missing or out of sync with `shard.yml` instead of silently
  re-resolving, and skips `development_dependencies`.
- **Availability = GitHub availability.** No registry means no registry
  mirror. A deleted or privated upstream repo breaks fresh installs
  immediately; teams with uptime requirements mirror their dependency set
  internally.
- **Small ecosystem, real abandonment rate.** Crystal's shard ecosystem is a
  few thousand packages, and a meaningful share are unmaintained. Check last
  activity on shardbox.org before adopting; forking-and-repointing a dead
  shard is routine Crystal practice, and dependency override support swaps
  sources without patching every transitive manifest.
- **Compiler coupling.** A dependency that compiles on Crystal 1.x may break
  on a newer 1.y as the stdlib evolves; the `crystal:` field in `shard.yml`
  does not fully protect you. Pin your Crystal version in CI alongside
  `shard.lock`.

## When to Use / When Not

**Use when:**
- You write Crystal. It is the official tool; there is no practical
  competitor, and fighting it buys nothing.
- You want git-native publishing: tagging a release is the whole workflow.
- You need reproducible builds — `shard.lock` plus committed `lib/` gives a
  fully self-contained source tree.

**Avoid when:**
- You need an immutable, mirrored registry with namespacing and yank/audit
  tooling — really an argument about choosing Crystal, not shards, but a real
  gap versus Cargo or npm.
- You expected private-registry workflows (scoped tokens, proxies): private
  deps work via plain git auth (SSH keys, HTTPS credentials), and that is all
  you get.

## Alternatives

No alternative exists within Crystal; these are the equivalent tools you
would be choosing by choosing a different language:

- rust-lang/cargo — use instead when you want a registry (crates.io) with
  immutable artifacts, yanking, and integrated build tooling.
- rubygems/bundler — the closest design ancestor (same Molinillo lineage) with
  a central gem registry; Ruby's ecosystem is orders of magnitude larger.
- golang/go — Go modules share the decentralized VCS-based model but add a
  module proxy and checksum database, fixing the mutability/availability gaps
  shards still has.
- elixir-lang/elixir — Mix + Hex pairs a build tool with a registry (hex.pm),
  the model Crystal deliberately did not adopt.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015-05 | Initial release by Julien Portalier; repo created 2015-05-20. |
| ~0.6–0.7 | 2016 | Distributed with official Crystal releases; `shard.lock` workflow established. |
| 0.9.0 | 2019-06 | Resolver rewritten on a Crystal port of Molinillo[^3]. |
| 0.17.x+ | 2022– | Dependency override support; additional VCS resolvers; ongoing maintenance in lockstep with Crystal releases (latest push 2026-07). |

## References

[^1]: crystal-lang/shards README and `shard.yml` specification. https://github.com/crystal-lang/shards/blob/master/docs/shard.yml.adoc
[^2]: Shardbox — community database of Crystal shards. https://shardbox.org/
[^3]: Shards v0.9.0 release (Molinillo-based resolver). https://github.com/crystal-lang/shards/releases/tag/v0.9.0 ; Molinillo algorithm: https://github.com/CocoaPods/Molinillo

## Tags

crystal, package-manager, dependency-resolution, build-tooling, git-based, decentralized, cli, developer-tools
