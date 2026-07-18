# janestreet/base

> Jane Street's wholesale replacement for the OCaml standard library — portable, consistent, and opinionated enough to deprecate the stdlib out from under you.

[GitHub repo](https://github.com/janestreet/base) ·
[API docs](https://ocaml.org/p/base/latest) ·
[License: MIT](https://github.com/janestreet/base/blob/master/LICENSE.md)

## Overview

Base is an alternative standard library for OCaml, developed by Jane Street and extracted from their older, larger Core library in 2016[^1]. Unlike stdlib-extension projects, Base is designed as a *replacement*: `open Base` shadows the compiler's default environment and deprecates the stdlib modules, forcing you into Base's idioms. What survives the cut is deliberately narrow — general-purpose data structures and functions that are portable to any environment that can run OCaml code, which is why I/O is excluded entirely and left to companion libraries (Stdio, Core_unix, Async)[^1].

The defining tradeoff is consistency versus interop. Base rejects OCaml's polymorphic comparison (`=`, `compare` on arbitrary values), replaces polymorphic hashing, uses labeled arguments pervasively (`List.fold ~init ~f`), attaches s-expression serializers to nearly every type, and threads comparator witnesses through its Map/Set types. Code written against Base is more uniform and harder to misuse than stdlib code — but it reads differently from the rest of the OCaml ecosystem, and its container types are not the stdlib's, so boundaries with stdlib-based libraries need explicit conversion.

Adoption is significant relative to the OCaml community's size: Base is the foundation of the entire Jane Street open-source suite (Core, Async, hundreds of ppx libraries), and *Real World OCaml*, the most widely used OCaml book, teaches the language through Base rather than the stdlib[^2]. The GitHub repo (~1.1k stars) is an export of Jane Street's internal monorepo and is actively maintained — last pushed July 2026 — with releases following the suite's synchronized versioning.

## Getting Started

```bash
opam install base
```

Base's only build dependencies are dune and sexplib0; it has no runtime dependencies[^1]. In a dune project:

```lisp
(executable
 (name main)
 (libraries base stdio))
```

```ocaml
open Base
open Stdio

let () =
  let counts =
    List.fold [ "a"; "b"; "a" ]
      ~init:(Map.empty (module String))
      ~f:(fun acc key ->
        Map.update acc key ~f:(function None -> 1 | Some n -> n + 1))
  in
  Map.iteri counts ~f:(fun ~key ~data -> printf "%s: %d\n" key data)
```

Note the shape: labeled arguments, a first-class module `(module String)` supplying the comparator, and `Stdio` for printing because Base itself has no I/O. After `open Base`, the original stdlib remains reachable as `Stdlib.String`, `Stdlib.print_string`, etc.[^1]

## Architecture / How It Works

**Shadowing, not extending.** `open Base` replaces the default namespace. Comparison operators become integer-only (`=` is `int -> int -> bool`); comparing other types requires type-specific functions (`String.equal`, `List.compare String.compare`) or an explicit opt-in via the `Poly` module. This turns a notorious class of OCaml bugs — structural polymorphic comparison doing the wrong thing on functions, cyclic values, or custom types — into compile errors[^1].

**Comparator witnesses.** Base's `Map` and `Set` carry a phantom type parameter identifying the comparison function used to build them: `('k, 'v, 'cmp) Map.t`, written in signatures as `Map.M(String).t`. Two maps built with different comparators are different types, so you cannot accidentally merge them. The cost is ceremony: every empty map needs `(module String)`-style first-class module arguments, and the types confuse newcomers.

**Committed codegen.** Base relies on ppx-derived code for comparison, hashing, and sexp conversion, but does not require ppx to build. Generated code is checked into the source between `[@@deriving_inline sexp_of]` and `[@@@end]` markers, with ppx used as a verification tool during development — similar to expectation testing[^1]. This keeps the dependency footprint minimal and bootstrap-friendly.

**Layering.** Internally, modules that exist in both Base and the stdlib (e.g. `String`) are built atop `String0`-style shim modules to break dependency cycles, and opening `Stdlib` inside Base is banned by a custom linter (`ppx_base_lint`)[^1]. Externally, Base sits at the bottom of a strict stack: Base → Core (adds heavier data structures, time, bignum interop) → Core_unix / Async (OS and concurrency). Portability is the design constraint that draws the line — Base works under js_of_ocaml and on Windows because nothing in it touches the OS.

**Sexps everywhere.** Nearly every type ships `sexp_of_t` / `t_of_sexp` via the sexplib0 dependency, making s-expressions the ecosystem's default debugging and serialization format. Note the trap the README itself documents: `String.sexp_of_t` embeds a string *in* a sexp, while `Sexp.of_string` parses the textual form of a sexp — same types, different meaning[^1].

## Production Notes

**Lockstep versioning is the big operational constraint.** Jane Street releases its entire open-source suite (base, core, ppx_*, async, ~100+ packages) as synchronized `v0.N` waves, roughly annually[^3]. Mixing versions across the suite is unsupported. In practice this means: upgrading Base means upgrading everything Jane Street at once; a third-party library pinned to an old `v0.N` can block your whole tree in the opam solver; and the `0.x` numbering is honest — each wave contains breaking API changes with no semver-style compatibility promise.

**`open Base` changes semantics silently at the source level.** Porting stdlib code under `open Base` typically fails to compile (good), but the failure modes are unintuitive to newcomers: `=` on strings becomes a type error, `Hashtbl.create` has a different signature, `List.mem` takes `~equal`. Budget real migration time; mechanical porting is not possible.

**Container types don't interoperate with stdlib containers.** Base reuses primitive stdlib types (`list`, `option`, `string`, `array`), so simple values flow freely. But `Base.Map` / `Base.Set` / `Base.Hashtbl` are distinct types from `Stdlib.Map` / `Set` / `Hashtbl`. Libraries written against the stdlib force conversion code at API boundaries — the practical cost of the two-ecosystems split in OCaml (Base world vs. stdlib world).

**Contribution friction.** Development happens in Jane Street's internal monorepo; the GitHub repo is a public export. External PRs are accepted but are imported through internal review, so turnaround can be slow, and history occasionally arrives as bulk sync commits. The low open-issue count (single digits) reflects this workflow as much as project health.

**Doc discovery.** Base's API is large and its docs are hosted per-version on ocaml.org[^4]; odoc-generated pages are accurate but terse. *Real World OCaml* is the de facto tutorial layer[^2].

**Post-2024 direction.** Jane Street's OxCaml compiler extensions (modes, stack allocation) have begun surfacing in the public suite's source as annotations; upstream OCaml compatibility is preserved, but expect the code to look increasingly unfamiliar if you read Base's internals[^5].

## When to Use / When Not

**Use when:**
- You're building on any Jane Street library (Core, Async, Incremental, Bonsai) — Base is non-negotiable there.
- You want stricter defaults than the stdlib: no polymorphic compare, labeled arguments, uniform module interfaces.
- You need one library codebase to run natively, under js_of_ocaml, and on Windows — Base's portability line is exactly this.
- You're following *Real World OCaml* — the book assumes it.

**Avoid when:**
- Your dependencies are stdlib-flavored (Lwt ecosystem, Mirage, much of opam) — you'll pay conversion costs at every boundary.
- You need version stability across years — annual breaking `v0.N` waves are the norm.
- You're writing a small library for broad consumption — depending on Base forces the choice onto all your users; stdlib-only or containers is more neutral.
- Binary size or dependency minimalism matters more than API ergonomics.

## Alternatives

- ocaml/ocaml — the stdlib itself has improved substantially since 4.07 (Seq, Result, Option modules); use it when you want zero-dependency neutrality.
- c-cube/ocaml-containers — extends the stdlib instead of replacing it; use when you want extra functions without leaving stdlib idioms.
- janestreet/core — Base's superset; use when you also need time, bignum, heavier containers, and don't care about js_of_ocaml portability.
- ocaml-batteries-team/batteries-included — the older comprehensive stdlib extension; mostly of historical interest now, but stdlib-compatible.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2016-09 | Repo created; Base extracted from Core_kernel as a portable minimal stdlib[^1]. |
| v0.9 | 2017 | First stable opam release[^3]. |
| v0.14 | 2020 | Suite wave; Base API largely settled into current shape[^3]. |
| v0.15 | 2022 | Suite reorganization: Core_kernel folded into Core; Base remains the portable bottom layer[^3]. |
| v0.16 | 2023 | Suite wave; OCaml 5.x compatibility work across the stack[^3]. |
| v0.17 | 2024 | Suite wave; continued OCaml 5 support[^3]. |

## References

[^1]: Base README. https://github.com/janestreet/base#readme
[^2]: Yaron Minsky, Anil Madhavapeddy, Jason Hickey, *Real World OCaml* (2nd ed.) — uses Base/Core throughout. https://dev.realworldocaml.org/
[^3]: base release history on opam. https://opam.ocaml.org/packages/base/
[^4]: Base API documentation. https://ocaml.org/p/base/latest
[^5]: OxCaml — Jane Street's OCaml extensions. https://oxcaml.org/

## Tags

ocaml, standard-library, stdlib-replacement, functional-programming, jane-street, sexp-serialization, dune, portability, core-ecosystem
