# ocaml/opam

> The OCaml package manager — source-based, solver-driven, with per-project
> compiler switches and a Git-native development workflow.

[GitHub repo](https://github.com/ocaml/opam) ·
[Official website](https://opam.ocaml.org) ·
[License: LGPL-2.1 with linking exception](https://github.com/ocaml/opam/blob/master/LICENSE)

## Overview

Opam is the package manager of the OCaml ecosystem, created and maintained by
OCamlPro since 2012, with 1.0 shipping in March 2013[^1]. It is source-based:
there are no binary packages. Every install compiles from source against the
exact compiler and dependency versions in your current *switch* — an isolated
prefix holding one compiler plus its packages. Package metadata lives in a
plain Git repository (ocaml/opam-repository), which makes publishing a pull
request rather than an upload, and makes the whole registry forkable.

The defining design choice is that dependency resolution is a real constraint
problem solved by a real solver. Opam encodes install requests as CUDF (the
format from the Mancoosi research project) and hands them to a vendored
MILP-based solver (mccs), rather than the greedy or backtracking resolution
most language package managers use[^2]. The payoff is correct handling of
complex version constraints and multiple compiler lines; the cost is that
solutions can be non-obvious and conflict messages take practice to read.

The tradeoff to understand up front: source-based + solver-driven means opam
optimizes for correctness and flexibility over install speed. A fresh switch
compiles the OCaml compiler itself (minutes), and there is no official binary
cache. Teams coming from npm or cargo find the first hour slower and the
fifth year more predictable.

## Getting Started

```bash
# Linux/macOS (Windows is natively supported since 2.2)
bash -c "sh <(curl -fsSL https://opam.ocaml.org/install.sh)"
opam init                  # creates ~/.opam, fetches opam-repository
eval $(opam env)           # loads the switch environment into this shell
opam switch create 5.3.0   # builds an OCaml 5.3.0 compiler from source
opam install dune utop ocaml-lsp-server
```

Per-project workflow with a local switch:

```bash
cd my-project/
opam switch create . --deps-only     # switch lives in ./_opam
opam install . --deps-only --with-test
opam pin add mylib git+https://github.com/me/mylib   # track a dev branch
```

## Architecture / How It Works

- **Switches.** A switch is a self-contained prefix (compiler + packages +
  environment). Global switches live under `~/.opam/<name>`; local switches
  live in `./_opam` next to the project, giving per-project isolation without
  containers. Activation is environment-variable based: `eval $(opam env)` or
  an installed shell hook rewrites `PATH` and OCaml-specific variables.
- **Solver.** Requests are compiled to CUDF and solved by the vendored mccs
  solver (external solvers like aspcud remain pluggable). Since 2.0 no
  external solver install is required[^2]. Version ordering follows the
  Debian convention, not SemVer — opam's own releases use SemVer, but package
  version comparison does not assume it.
- **Repositories.** A repository is a tree of `opam` files (a custom, typed
  format parsed by opam-file-format) served over Git or HTTP tarball. The
  canonical one is ocaml/opam-repository, curated by human maintainers with
  CI that builds new releases against reverse dependencies.
- **Pinning.** `opam pin` overrides a package's source with a local directory
  or Git URL, re-resolving dependencies from the pinned metadata. This is the
  core dev loop: pin your work-in-progress library, and dependents build
  against it as if released.
- **Sandboxed builds.** Since 2.0, build and install commands run sandboxed
  (bubblewrap on Linux, sandbox-exec on macOS) with restricted filesystem
  access, so a package build cannot scribble outside its build directory[^3].
- **Depexts.** Opam packages declare external system dependencies (e.g.
  `libssl-dev`) and opam drives the host package manager (apt, brew, etc.) to
  install them — integrated into the core since 2.1, previously the separate
  opam-depext plugin[^4].

Opam deliberately does not build your code: that is dune's job. The two are
coupled in practice — dune-project files can generate opam metadata — but they
remain separate tools with separate release cycles.

## Production Notes

- **The environment model is the #1 footgun.** Forgetting `eval $(opam env)`
  (or stale shells after `opam switch`) leaves you compiling against the
  wrong switch. Install the shell hook at `opam init` and let it manage this.
- **Switch creation is expensive.** Each new switch builds the compiler from
  source. In CI, cache `~/.opam` (the `setup-ocaml` GitHub Action does this);
  locally, expect minutes per switch. There is no official binary package
  channel; OCamlPro's opam-bin exists as a third-party binary-cache layer.
- **No lockfile by default.** Plain `opam install` re-solves every time, so
  fresh environments can drift as opam-repository moves. `opam lock`
  (integrated since 2.1) pins exact versions; use it for reproducible CI[^4].
- **Repository metadata is mutable.** opam-repository maintainers add upper
  bounds to *already-released* packages when new incompatibilities surface.
  This self-heals the ecosystem, but means the solver can pick different
  versions next week without anyone publishing anything — another argument
  for lock files in production.
- **Sandboxing breaks in minimal containers.** bubblewrap needs kernel
  namespace support; inside Docker the standard workaround is
  `opam init --disable-sandboxing`. Know that this removes the isolation.
- **Windows before 2.2 was a fork.** Native Windows users lived on the
  unofficial opam-repository-mingw fork for years; 2.2 (July 2024) unified
  Windows support into mainline[^5]. Pre-2.2 blog posts and Stack Overflow
  answers about Windows setup are obsolete.
- **Solver conflicts are terse.** Error output has improved since 2.x but
  diagnosing "no solution" on large dependency cones still often means
  manually bisecting constraints with `opam install pkg --show-actions`.

The project is actively maintained (pushed July 2026), with roughly 1.4k
stars and 400 forks — small numbers that understate its position: it is the
default distribution channel for effectively all open-source OCaml.

## When to Use / When Not

**Use when:**
- You write OCaml. It is the ecosystem-standard package manager and the only
  route to opam-repository's curated package set.
- You need several compiler versions side by side (e.g. testing a library
  across OCaml 4.14 and 5.x) — switches make this trivial.
- You develop against unreleased dependencies — pinning Git branches is a
  first-class workflow, not a hack.

**Avoid when:**
- You want hermetic, binary-cached, cross-language builds — Nix subsumes
  opam's role at higher setup cost.
- Your team wants an npm-style project-local, lockfile-first workflow with
  zero global state — dune's package management preview or esy target this.
- You need fast cold-start installs in ephemeral environments and cannot
  cache `~/.opam` — source-only builds will dominate your CI time.

## Alternatives

- ocaml/dune — the OCaml build system; its package-management preview
  (built on opam-repository metadata) aims at a lockfile-first dev workflow
  and may become the front-end most developers touch. Use it instead when
  you want one tool for build + deps and accept preview-stage maturity.
- esy/esy — npm-style sandboxed builds with project lockfiles for
  OCaml/Reason. Use it instead for a JS-developer-familiar workflow,
  accepting a much smaller community.
- NixOS/nixpkgs — reproducible, binary-cached, cross-language. Use it
  instead when OCaml is one part of a polyglot deployment and you already
  pay the Nix learning curve.
- OCamlPro/opam-bin — not a replacement but a binary-package layer on top of
  opam. Use it when switch-creation time is your bottleneck.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2013-03 | First stable release, by OCamlPro[^1]. |
| 1.2.0 | 2014-10 | Developer-workflow release: richer pinning, `opam source`. |
| 2.0.0 | 2018-09 | Major rewrite: local switches, sandboxed builds, vendored mccs solver, `opam` files in source trees[^3]. |
| 2.1.0 | 2021-08 | Depexts integrated into core, lock files, CLI versioning[^4]. |
| 2.2.0 | 2024-07 | Native Windows support unified into mainline[^5]. |
| 2.3.0 | 2024-11 | Maintenance release: UX and performance fixes[^6]. |

## References

[^1]: opam project README and history; created 2012, maintained by OCamlPro. https://github.com/ocaml/opam
[^2]: opam developer manual — package format, solver, and version ordering. https://opam.ocaml.org/doc/Manual.html
[^3]: "opam 2.0.0 release" — opam blog, 2018. https://opam.ocaml.org/blog/opam-2-0-0/
[^4]: "opam 2.1.0 release" — opam blog, 2021. https://opam.ocaml.org/blog/opam-2-1-0/
[^5]: "opam 2.2.0 release" — opam blog, 2024. https://opam.ocaml.org/blog/opam-2-2-0/
[^6]: "opam 2.3.0 release" — opam blog, 2024. https://opam.ocaml.org/blog/opam-2-3-0/

## Tags

ocaml, package-manager, dependency-solver, source-based, cli, developer-tools, functional-programming, sandboxing
