# commercialhaskell/stack

> A build tool for Haskell built around curated, reproducible package snapshots — the "it just works" answer to cabal hell.

[GitHub repo](https://github.com/commercialhaskell/stack) ·
[Official website](https://haskellstack.org) ·
[License: BSD-3-Clause](https://github.com/commercialhaskell/stack/blob/master/LICENSE)

## Overview

Stack is a project-oriented build tool for Haskell, first released by FP Complete in June 2015[^1]. It was created in direct response to "cabal hell" — the era when `cabal-install` used a single mutable global package database and dependency solving routinely broke working installs. Stack's answer was to sidestep solving entirely: instead of asking a solver to find compatible versions, a project pins a **Stackage snapshot** — a curated set of thousands of Hackage packages tested to build together against one specific GHC version[^2]. The same `stack.yaml` builds the same way on any machine, years later, because the snapshot is immutable.

Stack also owns the whole toolchain: it downloads and installs the correct GHC for the snapshot, bundles MSYS2 on Windows, and keeps every project's builds isolated. For a newcomer, `stack new && stack build && stack test` works on Linux, macOS, and Windows without separately installing a compiler. That end-to-end ownership was radical in 2015 and split the Haskell community into stack and cabal camps for years; since then `cabal-install`'s Nix-style builds (v2, default since Cabal 3.0 in 2019) and the GHCup toolchain installer have closed most of the original gap, and the two tools now coexist rather than compete on pain avoidance.

Governance shifted after FP Complete stepped back from day-to-day development; the project has been community-maintained for several years, currently led by Mike Pilgrem[^3]. At ~4.1k stars, ~850 forks, and pushes as recent as this week, it is modestly starred for its importance — a large share of Haskell projects and CI pipelines still build with it — and actively maintained, with 609 open issues reflecting a long tail of platform and edge-case reports rather than abandonment.

## Getting Started

```bash
curl -sSL https://get.haskellstack.org/ | sh   # or: ghcup install stack
stack new my-project
cd my-project
stack build        # downloads GHC for the snapshot on first run
stack test
stack exec my-project-exe
```

Project configuration lives in `stack.yaml`:

```yaml
# stack.yaml
snapshot: lts-23.0        # pins GHC + thousands of package versions
extra-deps:
  - some-pkg-1.2.3        # Hackage packages not in the snapshot
```

Stack also runs standalone scripts with pinned dependencies — a shebang line makes a `.hs` file reproducibly executable:

```haskell
#!/usr/bin/env stack
-- stack script --snapshot lts-23.0 --package turtle
import Turtle
main = echo "hello"
```

## Architecture / How It Works

**Snapshots, not solving.** A snapshot (Stackage LTS or Nightly) maps a GHC version plus exact versions of ~3,000 packages known to build together. Version resolution is a lookup, not a constraint problem. Anything outside the snapshot must be listed explicitly as an `extra-dep` — deliberate friction that keeps builds deterministic.

**Pantry.** Since the 2.x rewrite (2019), package identity is content-addressed: every package and file tree is referenced by SHA256, so `stack.yaml` + lock file pin dependencies down to bytes, not just version numbers[^4]. Archives are fetched from Hackage or the Casa content-addressed storage server. This is stronger reproducibility than a cabal freeze file, which pins versions but not content.

**Cabal underneath.** Stack is not a Cabal replacement at the build layer — it drives the Cabal *library* (`Setup.hs`, `build-type`) to compile each package, and every package still has a `.cabal` file. What Stack replaces is the `cabal-install` workflow: solving, fetching, database management, toolchain. It also integrates hpack, so projects can author `package.yaml` and have the `.cabal` file generated.

**Database layering.** Compiled artifacts split across three package databases: global (ships with GHC), snapshot (shared in `~/.stack` across all projects on the same snapshot), and local (per-project `.stack-work`). Building a heavy dependency once benefits every project on that snapshot — and dodges the classic mutable-global-db corruption that motivated the tool.

**Toolchain management.** `stack setup` installs GHC into `~/.stack/programs`, keyed by version; switching a project's snapshot switches its compiler without touching other projects. Docker and Nix integrations can substitute the entire build environment for teams that need bit-level system reproducibility.

## Production Notes

- **Disk usage grows without bounds.** `~/.stack` accumulates one GHC (~2–4 GB installed) plus one compiled snapshot-package set per snapshot you have ever used, and each project keeps a `.stack-work`. There is no garbage collection; teams routinely find 30–100 GB in `~/.stack` and prune by hand. CI caching of `~/.stack` is near-mandatory — a cold LTS build of a mid-size project can take 30+ minutes compiling dependencies.
- **Snapshot lag is the price of curation.** LTS snapshots trail Hackage; a library released last week is not in your snapshot. Each such package (and transitively, its incompatible deps) goes into `extra-deps`, and a project chasing a few bleeding-edge libraries can end up with dozens of extra-deps — at which point you are doing manual dependency solving, the thing Stack was built to avoid. `stack config` commands and lock files help, but the friction is structural.
- **Rebuild sensitivity.** Changing `ghc-options`, flags, or the snapshot invalidates large swaths of compiled artifacts. Stack's dirtiness tracking is conservative; "why is stack rebuilding everything" is a perennial issue-tracker theme. Keep option changes rare and prefer per-target options.
- **Interop with the cabal world is one-way.** Stack's databases are invisible to plain `cabal-install` and vice versa; a contributor using cabal on a stack project (or the reverse) builds everything twice. Haskell Language Server auto-detects which tool a project uses via `hie.yaml`/heuristics, but mixed-tool monorepos remain awkward.
- **GHCup coexistence.** Modern setups often install Stack *via* GHCup and configure Stack to reuse GHCup-installed GHCs (avoiding duplicate multi-GB compilers). Without that hook, Stack downloads its own GHC per snapshot even if an identical one already exists on the machine.
- **Version numbering quirk.** Stack has twice skipped a major version in releases: the 2.x line began at 2.1.1 (2019) and the 3.x line at 3.1.1 (2024). Scripts that pattern-match on "x.0" releases will wait forever.

## When to Use / When Not

**Use when:**
- You want reproducible Haskell builds with zero solver interaction — pin one snapshot line, done.
- You are onboarding newcomers or supporting Windows: Stack installing GHC + MSYS2 itself is still the smoothest cold-start path.
- Your CI must produce identical builds for years (content-addressed lock files pin bytes, not just versions).
- You maintain many projects and want compiled dependencies shared per-snapshot instead of rebuilt per-project.

**Avoid when:**
- You track bleeding-edge Hackage releases — snapshot lag turns into extra-deps bookkeeping, and cabal's solver handles that world better.
- You publish libraries and need to test against wide version ranges; cabal's solver-driven builds map better to Hackage's ecosystem-wide compatibility model.
- Disk space is constrained (CI runners, containers) and you cannot cache `~/.stack`.
- Your team already standardizes on GHCup + cabal — since Cabal 3.0 the reproducibility gap is small, and running both tools side by side doubles build artifacts.

## Alternatives

- haskell/cabal — the ecosystem-default build tool; use instead when you want solver-driven version selection, Hackage-first library development, or minimal tool divergence from upstream Haskell docs.
- haskell/ghcup-hs — toolchain installer (GHC, cabal, HLS, and Stack itself); use for compiler management when you don't want Stack owning GHC installs.
- NixOS/nixpkgs — Haskell package set inside Nix; use when you need reproducibility across the *entire* system environment, not just Haskell deps.
- input-output-hk/haskell.nix — Nix-based alternative build infrastructure translating cabal/stack projects into Nix derivations; use for hermetic, cached, cross-compiling CI at the cost of Nix expertise.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015-06 | Initial public release by FP Complete[^1]. |
| 1.0.0 | 2015-12 | First stable release. |
| 2.1.1 | 2019-06 | Pantry rewrite: content-addressed packages, lock files; skipped 2.0[^4]. |
| 2.x line | 2019–2024 | Incremental releases (2.3 → 2.15); GHC 9.x support, `snapshot` terminology. |
| 3.1.1 | 2024-07 | First 3.x release (skipped 3.0); skipped-version pattern repeats. |
| 3.x line | 2024– | Current line; active releases, repo pushed within days as of 2026-07. |

## References

[^1]: FP Complete, "stack: a new Haskell tool" announcement — 2015-06. https://www.fpcomplete.com/blog/2015/06/announcing-first-public-beta-stack/
[^2]: Stackage — stable Hackage package sets (LTS Haskell / Nightly). https://www.stackage.org/
[^3]: Stack repository maintainer activity and release signing, 2022–2026. https://github.com/commercialhaskell/stack/releases
[^4]: Stack 2.1.1 release notes (pantry, lock files). https://github.com/commercialhaskell/stack/blob/master/ChangeLog.md

## Tags

haskell, build-tool, package-manager, reproducible-builds, developer-tools, stackage, toolchain-management, dependency-management, cli
