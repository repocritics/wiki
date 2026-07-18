# janestreet/core

> Jane Street's industrial-strength standard library overlay for OCaml — the stdlib the largest industrial OCaml user actually runs on.

[GitHub repo](https://github.com/janestreet/core) ·
[Documentation](https://ocaml.janestreet.com/ocaml-core/latest/doc/core/index.html) ·
[License: MIT](https://github.com/janestreet/core/blob/master/LICENSE.md)

## Overview

Core is a full replacement for OCaml's standard library, developed and used in production by Jane Street, the trading firm that is the largest industrial user of OCaml[^1]. It is not an add-on collection of utilities: `open! Core` shadows the entire stdlib namespace and substitutes Jane Street's conventions — labeled arguments everywhere, `t`-first module signatures, `Or_error` for fallible operations, s-expression serializers on nearly every type, and a deliberate restriction of polymorphic comparison (`=` is redefined to work on `int` only; you use `String.equal`, `Float.equal`, etc. explicitly).

Core sits in a layered family: **Base** is the minimal, dependency-light, highly stable portable subset; **Core** extends Base with `bin_io` binary serialization, `Stable` versioned wire types, richer containers, time handling, and Quickcheck; Unix system calls live in the separate `core_unix` package[^2]. Core itself is portable and works under JavaScript via js_of_ocaml. This layering is the resolution of a decade of reshuffling — until v0.15 (2022) the portable subset was a separate library called `Core_kernel`, which was then folded into Core while the Unix-dependent code moved out to `core_unix`[^3].

The defining tension: Core is exported from Jane Street's internal monorepo, in lockstep with its sibling packages, on Jane Street's schedule and for Jane Street's needs. You get code hardened by a demanding production user, but the project is not community-governed — releases are roughly annual, APIs change between them without semver guarantees, and external PRs are imported manually rather than merged directly. At 1.3k stars it looks modest by GitHub metrics, but within the small OCaml ecosystem it is one of the most-depended-on opam packages and the library that Real World OCaml, the standard OCaml book, teaches with[^4].

## Getting Started

```bash
opam install core
# for the ppx derivers Core code conventionally uses:
opam install ppx_jane
```

```lisp
; dune
(executable
 (name main)
 (libraries core)
 (preprocess (pps ppx_jane)))
```

```ocaml
(* main.ml *)
open! Core

let word_counts words =
  List.fold words ~init:String.Map.empty ~f:(fun acc word ->
    Map.update acc word ~f:(function None -> 1 | Some n -> n + 1))

let () =
  word_counts [ "a"; "b"; "a" ]
  |> Map.iteri ~f:(fun ~key ~data -> printf "%s: %d\n" key data)
```

Note the house style: labeled arguments (`~init`, `~f`), comparator-carrying maps (`String.Map`), and `open! Core` at the top of every file.

## Architecture / How It Works

The repository is a multi-library export, not a single package: alongside `core/` it carries `command` (CLI parsing), `validate`, `heap_block`, `filename_base`, and others, each its own opam package cut from the same source drop.

- **Base extension, not a fork.** Most Core modules are extensions of their Base equivalents: the Core version of `Map`, `String`, or `Int` re-exports the Base implementation and layers on `bin_io`, `Stable` versions, and comparator conventions[^2]. If you only need containers and combinators, Base alone is a lighter dependency.
- **ppx-driven.** Core's ergonomics assume the `ppx_jane` deriver set: `[@@deriving sexp, compare, hash, bin_io, equal]` appears on virtually every type. Without the ppx toolchain much of the value (free serializers, comparators, Quickcheck generators) disappears, and build times pay for the preprocessing.
- **`Stable` modules.** Types intended for persistence or wire protocols get a `Stable.V1`, `V2`, ... submodule whose serialization format is frozen forever. This is the mechanism that lets Jane Street evolve APIs aggressively while keeping on-disk and on-wire data readable — API stability and format stability are deliberately decoupled.
- **Comparator witnesses.** `Map` and `Set` carry a phantom "comparator witness" type parameter, so a map built with `String.comparator` is type-distinct from one built with a custom ordering. Safer than the stdlib's polymorphic-compare maps; also the single biggest source of confusing type errors for newcomers.
- **Release model.** All `janestreet/*` packages share one version number (v0.17 at present) and are released together, typically once a year, from an internal snapshot. There is no public development-branch workflow in the usual sense; `master` moves when Jane Street exports.

## Production Notes

**Version lockstep is real.** You cannot mix `core.v0.16` with `ppx_jane.v0.17` — the entire Jane Street universe must resolve to one version series, and opam's solver will fight you if any transitive dependency pins a different one. Upgrading Core means upgrading every Jane Street package in your tree at once, and the annual releases routinely rename or remove functions (deprecations do appear one release ahead, e.g. v0.17 deprecated `Map.comparator` in favor of `Map.Comparator.Module.t`[^5]).

**No semver.** Treat every 0.N bump as a major release. `Stable` submodules protect your serialized data; nothing protects your call sites.

**Binary size and build time.** `open! Core` pulls in a large dependency cone (bin_prot, sexplib, time handling, Unicode tables). CLI tools that would be small stdlib binaries come out several MB, and cold builds are noticeably slower than Base-only or stdlib projects. Library authors in the OCaml community usually target Base (or the stdlib) precisely to avoid imposing Core on downstream users.

**Unix code is elsewhere.** Anything touching syscalls, `Unix.fork`, signals, or filenames-as-paths migrated to `core_unix` in v0.15[^3]. Old blog posts and Stack Overflow answers referencing `Core.Unix` or `Core_kernel` will not compile against modern Core — a persistent trap given how much OCaml documentation predates the split. `core_unix` does not support Windows; Core itself is portable, including JavaScript targets, and since v0.17 the timezone database works in browsers without bundling tzdata[^5].

**Polymorphic compare removal will bite once.** Code that used stdlib `=` on strings or lists fails to type-check after `open! Core`. This is by design (polymorphic compare on abstract types is a classic OCaml bug source), but budget for the migration on existing codebases.

Maintenance is active: the repo sees regular pushes (latest July 2026) and tracks OCaml 5 runtime changes in `Gc` and friends[^5], though OCaml 5 multicore idioms are still arriving incrementally.

## When to Use / When Not

**Use when:**
- You are building OCaml applications (not libraries) and want one coherent, production-proven stdlib with serialization, time, and containers solved.
- You need versioned binary/s-expression wire formats — `Stable` + `bin_io` is the most battle-tested answer in the ecosystem.
- You are working through Real World OCaml or joining a Jane Street-style codebase where Core idioms are the house dialect.

**Avoid when:**
- You are publishing a library for general OCaml consumption — depend on Base or the stdlib so you don't force the Core universe on users.
- Binary size or build time is tight (small CLIs, constrained targets).
- You need Windows and Unix syscalls — `core_unix` won't follow you there.
- You want a stdlib that values API stability over evolution; the stdlib itself has absorbed many ideas (`Seq`, `Result`, labeled modules) and changes far more slowly.

## Alternatives

- ocaml/ocaml — the built-in stdlib; use it when you want zero dependencies and maximum ecosystem compatibility; it has improved substantially since Core's founding era.
- janestreet/base — use instead when writing libraries or when you want Jane Street idioms without bin_io, time, and the heavy dependency cone.
- c-cube/ocaml-containers — use when you want stdlib-compatible extensions that augment rather than shadow the standard library.
- ocaml-batteries-team/batteries-included — the older community-driven "extended stdlib"; use when you prefer stdlib conventions over Jane Street's labeled/t-first style.

## History

| Version | Date | Notes |
|---------|------|-------|
| early releases | 2013 | Repo public 2013-01; internal date-derived version scheme (108.x–113.x). |
| 113.33.03 | 2016-04 | Last release of the old numbering scheme. |
| v0.9.0 | 2017-03 | Switch to unified vN versioning across all Jane Street packages; Base era begins. |
| v0.12.0 | 2019-02 | Steady annual-cadence releases established. |
| v0.14.0 | 2020-05 | — |
| v0.15.0 | 2022-02 | Core_kernel folded into Core; Unix code split to core_unix[^3]. |
| v0.16.0 | 2023-05 | — |
| v0.17.0 | 2024-05 | Unicode submodules (`Utf8`/`Utf16*`/`Utf32*`), OCaml 5 `Gc` accommodations, browser-usable timezones[^5]. |

## References

[^1]: Core README — "Portable standard library for OCaml". https://github.com/janestreet/core/blob/master/core/README.md
[^2]: Core README, "Relationship to Core and Base". https://github.com/janestreet/core/blob/master/core/README.md
[^3]: janestreet/core_unix — Unix-specific portions of Core, split out at v0.15. https://github.com/janestreet/core_unix
[^4]: Real World OCaml (2nd ed.) — uses Core throughout. https://dev.realworldocaml.org/
[^5]: Core CHANGES.md, v0.17.0 release notes. https://github.com/janestreet/core/blob/master/CHANGES.md

## Tags

ocaml, standard-library, stdlib-replacement, jane-street, serialization, bin-io, sexp, functional-programming, containers, js-of-ocaml
