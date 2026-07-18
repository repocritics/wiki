# nim-lang/nimble

> The default package manager for the Nim programming language — a decentralized, git-backed installer where the "registry" is a single JSON file of URLs.

[GitHub repo](https://github.com/nim-lang/nimble) ·
[Official docs](https://nim-lang.github.io/nimble/index.html) ·
[License: BSD-3-Clause](https://github.com/nim-lang/nimble/blob/master/license.txt)

## Overview

Nimble is the package manager that ships with the Nim compiler. It was started by Dominik Picheta in January 2011 under the name "babel" and renamed to Nimble in late 2014, alongside the language's own rename from Nimrod to Nim[^1][^2]. It handles the full package lifecycle: scaffolding (`nimble init`), dependency installation, building binaries, running tests and custom tasks, and publishing.

Its defining design decision is that there is no hosted registry. The package index is `packages.json` in the nim-lang/packages repository — a flat list mapping package names to git (or mercurial) URLs[^3]. Installing a package clones its repository and checks out a version tag; there are no server-hosted tarballs, no namespacing, and historically no immutable artifacts. This keeps infrastructure cost near zero and publishing friction low (publishing is a PR to a JSON file), at the price of npm/crates.io-style guarantees: tags are mutable, names are first-come-first-served, and a deleted upstream repo breaks installs. Lock files with checksums (added in the 0.14 line) are the mitigation.

At ~1.4k stars and ~210 forks the repo looks small, but the number understates reach: Nimble is bundled with every official Nim release, so effectively every Nim project touches it. Development is active (pushed July 2026), with ~250 open issues carried by a small maintainer team.

## Getting Started

Nimble ships with Nim; installing Nim (e.g. via choosenim) gives you both:

```bash
curl https://nim-lang.org/choosenim/init.sh -sSf | sh
nimble -v                 # version bundled with your Nim release
nimble init mypkg         # scaffold a package
nimble install karax      # install a package from the registry
nimble build              # build binaries declared in the .nimble file
nimble test               # run the test task
```

Package metadata lives in a `.nimble` file — evaluated as NimScript, so it mixes declarative fields with executable tasks:

```nim
# mypkg.nimble
version       = "0.1.0"
author        = "Your Name"
description   = "Example package"
license       = "MIT"
srcDir        = "src"
bin           = @["mypkg"]

requires "nim >= 2.0.0"
requires "jester >= 0.6.0"

task docs, "Generate documentation":
  exec "nim doc --project src/mypkg.nim"
```

## Architecture / How It Works

**Resolution.** `nimble install` fetches the package list, resolves the name to a URL, clones the repo, and picks a version by matching git tags against the version constraint. Dependencies are installed side-by-side into a global store (`~/.nimble/pkgs2`), multiple versions coexisting; Nimble then invokes `nim` with the right `--path` flags so the compiler sees exactly the resolved set. Recent versions replaced the older newest-satisfying heuristic with a SAT solver for dependency resolution[^4].

**NimScript evaluation.** Because `.nimble` files are code, reading a package's metadata means evaluating it with the Nim compiler in NimScript mode. This is the source of Nimble's two chronic complaints: cold resolution is slow (a compiler subprocess per manifest) and metadata parsing is arbitrary code execution. A declarative parser fast path for simple manifests has been added in recent versions to address the former[^4].

**Lock files.** `nimble lock` produces `nimble.lock` with exact versions, VCS revisions, and checksums; `nimble setup` materializes an environment from it. Before lock files landed (0.14 line), reproducible Nim builds required pinning by hand or avoiding Nimble entirely.

**Develop mode.** `nimble develop` clones dependencies into a working directory and links them for in-place editing — the workflow for patching a dependency while consuming it. The mechanism was reworked in 0.14 around develop files, and pre/post-0.14 documentation diverges accordingly[^5].

**Coupling to Nim.** Nimble's version is pinned to the Nim release that bundles it. Features described in the latest docs may be absent from the Nimble in your Nim toolchain — the docs themselves warn about this[^2]. The compiler in turn depends on Nimble's store layout (`--nimblePath`), so the two are co-evolved and upgraded together.

## Production Notes

- **Commit to lock files.** Without `nimble.lock`, builds float on mutable git tags in third-party repos. Tags can be moved or deleted upstream; the lock file's checksums are the only integrity guarantee in the system.
- **The global store leaks between projects.** All projects share `~/.nimble/pkgs2` by default. For isolation, create a `nimbledeps` directory in the project (or use `nimble install -l`) to switch to local dependency mode — the closest equivalent to node_modules.
- **Manifest evaluation is code execution.** `nimble install` runs NimScript from every package in the dependency tree at resolution time, before you have reviewed anything. Treat installing an unknown package like running its code, because it is.
- **Registry fragility.** Package URLs point at personal GitHub repos. Renames, deletions, and force-pushes upstream have broken installs before; there is no central mirror of artifacts. Vendoring or lock files are the defense.
- **CI and version drift.** Use `nimble install -y --depsOnly` and cache `~/.nimble` keyed on the lock file. Nimble behavior changes with the bundled Nim release, so pin the Nim version (choosenim or a container image) as strictly as the packages, and check `nimble -v` before relying on new flags.

## When to Use / When Not

**Use when:**
- You write Nim. It is the default, the ecosystem's packages are indexed for it, and every tutorial assumes it.
- You want near-zero-friction publishing: tag a release, PR one JSON line.
- You need per-project isolation and reproducibility — lock files plus `nimbledeps` cover this adequately today.

**Avoid when:**
- You need supply-chain guarantees comparable to crates.io or npm (immutable hosted artifacts, signed publishes, yanking) — the git-URL model cannot provide them.
- Your organization forbids running unreviewed code at install time and cannot sandbox the resolution step.
- You run large multi-repo Nim codebases and hit the limits of Nimble's resolution model — teams like Status have historically opted for submodule-based builds instead.

## Alternatives

- nim-lang/atlas — workspace-oriented dependency cloner distributed with Nim 2.x; use it when you prefer flat source checkouts under your control over a managed package store.
- disruptek/nimph — community package handler with stricter git-native semantics; largely dormant, use only if its model already fits your workflow.
- status-im/nimbus-build-system — make + git submodules harness used by the Status/Nimbus teams; use when you need fully deterministic multi-repo builds and accept the manual upkeep.
- git submodules alone — viable for small projects with few dependencies, at the cost of all tooling conveniences.

## History

| Version | Date | Notes |
|---------|------|-------|
| — (babel) | 2011-01 | Project created by Dominik Picheta as "babel"[^1]. |
| — | 2014-12 | Renamed babel → Nimble alongside Nimrod → Nim[^2]. |
| 0.14 | 2022 | Lock files with checksums, `nimble setup`, reworked `nimble develop`[^5]. |
| 0.16 | 2024 | SAT-solver dependency resolution; declarative parser work[^4]. |
| — | 2026-07 | Actively maintained; latest push 2026-07-15. |

## References

[^1]: GitHub repository metadata — created 2011-01-27; authorship per README. https://github.com/nim-lang/nimble
[^2]: Nimble Guide (official documentation). https://nim-lang.github.io/nimble/index.html
[^3]: nim-lang/packages — the central package index (`packages.json`). https://github.com/nim-lang/packages
[^4]: Nimble changelog. https://github.com/nim-lang/nimble/blob/master/changelog.markdown
[^5]: Nimble Guide, "Development workflow" (`nimble develop`). https://nim-lang.github.io/nimble/workflow.html

## Tags

nim, package-manager, dependency-resolution, build-tool, cli, nimscript, developer-tools, registry, supply-chain
