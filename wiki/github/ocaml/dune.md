# ocaml/dune

> The standard build system of the OCaml ecosystem — declarative S-expression
> configs, hermetic incremental builds, and Jane Street engineering DNA.

[GitHub repo](https://github.com/ocaml/dune) ·
[Official website](https://dune.build/) ·
[License: MIT](https://github.com/ocaml/dune/blob/main/LICENSE.md)

## Overview

Dune is the de facto build system for OCaml. It began life in 2017 as
**jbuilder**, an open-sourcing of the build scheme Jane Street used
internally, and was renamed Dune for its 1.0 release in 2018 after a name
conflict with the older JBuilder IDE[^1]. You describe libraries, executables,
and tests in small declarative `dune` files (S-expression syntax); Dune
computes the build graph, runs compilations in parallel with sandboxing and
content digests, and handles the OCaml-specific plumbing (module dependency
discovery, ppx preprocessing, odoc docs) that made predecessor tools painful.

Its GitHub numbers undersell it: ~1.9k stars, but essentially every actively
maintained OCaml package on opam builds with Dune — Jane Street's public
libraries and Coq included. Dune's actual position is "Cargo for OCaml, minus
the package manager", and that gap is the project's defining tension.
Historically Dune only *builds*; dependency management belongs to opam, a
separate tool with its own model, and the opam/dune split is the most common
newcomer confusion. The in-progress `dune pkg` mode (developer preview since
2024) aims to close that gap and make `dune` the only command needed[^2].

The other defining trait is deliberate inexpressiveness: `dune` files are
static data, not a scripting language — fast and reproducible, at the cost of
awkwardness whenever you need logic the DSL doesn't anticipate.

## Getting Started

```bash
opam install dune          # via the opam package manager
dune init proj hello       # scaffold bin/ lib/ test/ + dune-project
cd hello
dune build                 # build everything
dune exec hello            # build and run the executable
dune test                  # run the test stanza
```

A `dune-project` file (`(lang dune 3.16)`) pins the config language version —
old projects keep building unchanged under new Dune binaries. Stanzas in
`dune` files describe each buildable thing:

```lisp
;; bin/dune
(executable
 (public_name hello)   ; installed/opam-visible name
 (name main)           ; module containing the entry point
 (libraries hello))    ; depends on the sibling library in lib/
```

## Architecture / How It Works

Dune walks the workspace, reads every `dune` file, and elaborates stanzas into
a rule database mapping target paths to actions. Notable internals:

- **Versioned configuration language.** `(lang dune X.Y)` in `dune-project`
  selects behavior per project; compatibility shims mean a 2018 project still
  builds with a 2026 binary. This let the ecosystem upgrade without flag
  days[^3].
- **Incremental engine.** Rules re-execute on content digests, not
  timestamps. An internal incremental-computation library ("memo") memoizes
  rule elaboration itself, which is what makes watch mode (`dune build -w`)
  rebuild in fractions of a second on large trees.
- **Sandboxed actions and a shared cache.** Actions can run in sandboxes so
  undeclared dependencies fail loudly; a content-addressed shared cache
  (`dune cache`) deduplicates artifacts across workspaces[^1].
- **Composability.** Dropping multiple Dune projects into one directory tree
  yields a single build; `(vendored_dirs ...)` marks third-party source to
  build but not lint. This is what makes OCaml monorepos practical.
- **Wrapped libraries.** Each library gets a namespace module via generated
  module aliases, so two libraries can both contain a `Utils` module without
  clashing — a real problem in the pre-Dune flat-namespace world.
- **Toolchain integration.** Dune serves Merlin/ocamllsp configuration
  directly (`dune ocaml-merlin`), generates `.opam` files from `dune-project`
  metadata, builds odoc documentation (`dune build @doc`), and has first-class
  rules for js_of_ocaml, Melange (OCaml-to-JavaScript), and Coq theories.

The only escape hatch for custom logic is the `(rule ...)` stanza's small
action DSL — there is intentionally no way to run arbitrary OCaml at
configuration time.

## Production Notes

- **The opam/dune split is real onboarding friction.** `dune build` fails on a
  missing dependency with a compiler error, not "run opam install X". Teams
  script around it (`make deps`) or adopt `dune pkg` — which works but is
  still a developer preview and changes between minor releases[^2].
- **The DSL wall.** Code generation beyond simple rules (protobuf pipelines,
  version stamping, conditional compilation) turns into contortions with
  `(rule ...)`, `%{...}` variables, and generated included files — workable
  but poor-reading; budget time for nontrivial build logic.
- **Profiles change semantics.** The default `dev` profile enables extra
  warnings and lighter optimization; ship with `--profile release`.
- **ppx cost.** Preprocessor-heavy codebases (Jane Street-style `ppx_jane`
  stacks) pay a large compile-time cost; watch mode plus the cache mitigates.
- **Issue backlog.** ~990 open issues against a small core team[^4]. The tool
  is stable, but niche bug reports can sit for years; check existing issues
  before assuming a workaround is temporary.
- **Version skew.** A library raising its `(lang dune X.Y)` version forces
  contributors onto a newer Dune binary. Minor releases are frequent; pin
  Dune in CI.
- **Windows actually works.** Dune is self-contained, relocatable, and needs
  no system dependencies beyond OCaml — Windows Console included[^3]. Unusual
  and genuine in the OCaml world.

## When to Use / When Not

**Use when:**
- You write OCaml. This is not really a choice anymore — the ecosystem,
  editors, and opam tooling all assume Dune.
- You maintain a multi-package OCaml repo and want one build with
  `--only-packages` slicing for release.
- You need cross-compilation or builds against several OCaml versions at once
  (workspace contexts)[^5], or Coq/Melange targets in the same graph.

**Avoid when:**
- Your project is polyglot with OCaml as a minor component — Bazel or Nix
  composes better across languages than Dune does.
- Your build is mostly arbitrary codegen and shell logic; the static DSL will
  fight you where Make or a general build tool would not.
- You need a mature lockfile-based, single-tool dependency story today —
  `dune pkg` is promising but not yet the stable default.

## Alternatives

- ocaml/ocamlbuild — the previous standard, now maintenance-only; use only
  when maintaining legacy projects that never migrated.
- bazelbuild/bazel — use with obazl rules when OCaml is one language among
  many in a large polyglot monorepo.
- esy/esy — npm-style sandboxed project workflow for OCaml/Reason; use when
  you want per-project isolated dependency installs without opam switches.
- janestreet/jenga — Jane Street's earlier internal build system, superseded
  by Dune; historical interest only.
- ocaml/opam — the complement, not a competitor: package manager and
  compiler-switch tool that `dune pkg` may eventually absorb.

## History

| Version | Date | Notes |
|---------|------|-------|
| jbuilder | 2017 | First public release of Jane Street's build scheme[^1]. |
| 1.0 | 2018-07 | Renamed jbuilder → Dune; versioned `(lang dune ...)` config. |
| 2.0 | 2019 | Dropped legacy `jbuild` file support; stricter defaults. |
| 3.0 | 2022 | Current major line; improved watch mode and caching. |
| 3.x pkg | 2024 | `dune pkg` package management developer preview[^2]. |
| 3.x | 2026 | Development continues on the 3.x line[^4]. |

## References

[^1]: Dune manual — history, cache, and core concepts. https://dune.readthedocs.io/en/latest/
[^2]: Dune website — package management developer preview. https://dune.build/
[^3]: ocaml/dune README — Jane Street origin, relocatable binary, Windows support. https://github.com/ocaml/dune#readme
[^4]: GitHub API snapshot, 2026-07: 1,902 stars, 487 forks, 991 open issues, MIT, last push 2026-07-18. https://github.com/ocaml/dune
[^5]: Dune docs — cross-compilation via workspace contexts. https://dune.readthedocs.io/en/latest/cross-compilation.html

## Tags

ocaml, build-system, build-tool, incremental-build, monorepo, jane-street, developer-tools, compiler-tooling, package-management, functional-programming
