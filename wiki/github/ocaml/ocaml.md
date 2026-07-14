# ocaml/ocaml

> The core OCaml system — a statically-typed functional language from the ML family, with type inference, a heavyweight module system, and both bytecode and optimizing native compilers.

[GitHub repo](https://github.com/ocaml/ocaml) ·
[Official website](https://ocaml.org) ·
License: LGPL-2.1-with-linking-exception (GitHub reports `NOASSERTION`)

## Overview

OCaml is a general-purpose language descended from Caml, itself part of the ML lineage begun by Robin Milner. Development has happened at INRIA (France's national computing research institute) since the late 1980s; the current object-oriented incarnation, "Objective Caml," shipped in 1996 and was renamed simply "OCaml" in 2011[^1]. It combines Hindley-Milner type inference (you rarely write type annotations, yet everything is statically checked), algebraic data types with exhaustive pattern matching, a module system with functors (modules parameterized by other modules), and an object system that most users never touch.

The defining tradeoff is **pragmatism over purity**. Unlike Haskell, OCaml is strict (eager) by default and allows unrestricted side effects, mutation, and imperative loops anywhere. This makes it easy to reason about performance and to write imperative code where it is natural, at the cost of the referential-transparency guarantees a lazy pure language provides. The type system is expressive (GADTs, polymorphic variants, first-class modules) but stops short of dependent types, which keeps inference decidable and error messages tractable — though module-heavy code can still produce famously long type errors.

OCaml's largest industrial backer is Jane Street, which runs its trading systems on it and funds much of the compiler and tooling work[^2]. It is also the implementation language of the Rocq (formerly Coq) proof assistant, the Frama-C and Astrée static analyzers, the Tezos blockchain, MirageOS unikernels, and several Meta developer tools (Flow, Hack, Infer; ReasonML and the original Rescript compiler share the lineage). The community is small relative to mainstream languages but dense with compiler, formal-methods, and finance users.

## Getting Started

Install via **opam**, the OCaml package manager, which manages compiler versions ("switches") the way rustup or nvm manage toolchains:

```bash
bash -c "sh <(curl -fsSL https://opam.ocaml.org/install.sh)"
opam init
opam switch create 5.2.0     # create an isolated compiler + package sandbox
eval $(opam env)
opam install dune utop        # standard build tool + enhanced REPL
```

A minimal program, with type inference and pattern matching (no type annotations needed):

```ocaml
(* main.ml *)
type user = { id : int; name : string }

let greet u = Printf.sprintf "Hello, %s" u.name

let () =
  let users = [ { id = 1; name = "Tom" }; { id = 2; name = "Brad" } ] in
  List.iter (fun u -> print_endline (greet u)) users
```

Build and run with the standard tool:

```bash
dune init exe app .
dune exec ./main.exe
```

## Architecture / How It Works

The repository ships **two compilers that share a front end**:

1. **`ocamlc`** — compiles to bytecode interpreted by a C runtime. Fast compilation, compact output, portable to essentially any platform with a C compiler, and the only backend on some architectures.
2. **`ocamlopt`** — an optimizing native-code compiler emitting assembly for x86-64, ARM64, POWER, RISC-V, and IBM Z. Slower to compile, larger binaries, but performance competitive with C/C++ for many workloads.

Both front ends parse to a typed AST, run type inference and the module-system checks, then lower through a shared intermediate representation (Lambda). `ocamlopt` continues through Flambda (an optional, more aggressive inlining/optimization pass enabled with a dedicated switch variant) down to Cmm and platform-specific assembly.

**The garbage collector** is a generational, mostly-incremental collector. Historically it was a simple, low-pause single-threaded design that was a large part of OCaml's predictable-latency reputation. As of **OCaml 5.0 (December 2022)** the runtime was rewritten for shared-memory parallelism[^3]: computations run across **domains** (units of parallelism mapping to OS threads/cores), backed by a new parallel-aware GC. This was the culmination of the multi-year Multicore OCaml research effort.

5.0 also stabilized **effect handlers** — a delimited-continuation mechanism that lets libraries implement their own concurrency schedulers (the `eio` and `domainslib` libraries build on it) without `async`/`await` keywords baked into the language. Effects are currently *unchecked* by the type system, an acknowledged gap: a function's type does not tell you which effects it may perform.

Native `4.x` compilation supported 32-bit targets; **from 5.0 onward native compilation is 64-bit only**[^3], with no plans to restore 32-bit native support.

## Production Notes

**The 4.14 vs 5.x split is the central operational decision.** OCaml 4.14 (March 2022) is the final 4.x release and remains maintained precisely because the 5.x GC rewrite changed allocation and collection performance characteristics. The project explicitly asks maintainers to benchmark 5.x and report regressions[^4]. Some latency-sensitive codebases (Jane Street among them) migrated cautiously; expect to profile rather than assume 5.x is a drop-in speedup.

**Parallelism is real but coarse-grained.** Domains are heavyweight — you spawn a handful (roughly one per core), not thousands. Fine-grained concurrency (many I/O-bound tasks) is handled by effect-based schedulers like `eio` layered on top, not by spawning domains. Mixing the old `Thread` module, `Unix.fork`, domains, and effect schedulers in one process requires care; the ecosystem is still consolidating around post-5.0 patterns.

**The standard library is deliberately small and was long considered inadequate.** For years the community split between the stdlib, Jane Street's **Core**, and **Batteries**. Newer OCaml has absorbed many gaps, but you will still reach for third-party libraries (`containers`, `base`) for data structures and utilities that other languages ship in the box.

**Tooling has consolidated but the history leaks through.** `dune` is now the near-universal build system and `opam` the package manager, both mature. But older projects use `ocamlbuild`, `oasis`, or raw Makefiles, and several formerly-bundled libraries (`graphics`, `num`, `ocamlbuild`, `camlp4`, the `Stream` module) were **removed from the core distribution** and are now separate opam packages[^5] — a common source of "it compiled on 4.02" breakage.

**Windows is a second-class-but-improving target.** Native Windows builds exist (MSVC and MinGW flavors) but Unix/Linux/macOS are the primary platforms; many developers on Windows use WSL. opam's Windows support matured only relatively recently.

**Error messages.** Type inference means errors surface far from their cause, and functor/module errors can span dozens of lines. This is the most common early-stage complaint and a real onboarding cost.

## When to Use / When Not

**Use when:**
- You want a fast, statically-typed language with a GC and predictable performance (compilers, static analyzers, trading systems, proof assistants).
- Correctness matters and you want algebraic data types + exhaustive matching to make illegal states unrepresentable.
- You are doing programming-language or formal-methods work — OCaml is a lingua franca of that field.
- You want native single-binary output without a heavy runtime.

**Avoid when:**
- You need a large hiring pool or a big library ecosystem for a mainstream domain (web, mobile, ML) — the community is small.
- You want lazy/pure semantics and effect tracking in the type system (Haskell fits better).
- You need best-in-class Windows-native support or GUI toolkits.
- Your team cannot absorb the module-system and type-error learning curve for the project's timeline.

## Alternatives

- haskell/ghc — use instead when you want laziness, purity, and type-level effect tracking, and can accept harder performance reasoning.
- dotnet/fsharp — use instead when you want ML-style functional programming on a large runtime with a mainstream ecosystem (.NET) and Windows-first tooling.
- rust-lang/rust — use instead when you need memory safety without a GC and manual control over allocation for systems programming.
- scala/scala — use instead when you want functional + OO on the JVM with Java interop.
- Standard ML (MLton / SML/NJ) — use instead when you want a smaller, formally-specified ML core without OCaml's extensions.

## History

| Version | Date | Notes |
|---------|------|-------|
| Objective Caml 1.00 | 1996 | First release of the OO Caml system at INRIA[^1]. |
| 3.00 | 2000 | Major line; labeled/optional arguments, polymorphic variants matured. |
| 3.12 | 2010-08 | First-class modules, module aliases. |
| 4.00 | 2012-07 | GADTs; project renamed OCaml (from 2011). |
| 4.03 | 2016 | Flambda optimizer, inline records, relicensing to LGPL-2.1 era. |
| 4.14 | 2022-03 | Final release of the 4.x series; still maintained[^4]. |
| 5.0 | 2022-12 | Multicore runtime rewrite: domains, effect handlers; native 64-bit only[^3]. |
| 5.1 | 2023-09 | Post-multicore stabilization and performance work. |
| 5.2 | 2024-05 | Continued runtime and tooling improvements. |

## References

[^1]: OCaml history and the Caml/Objective Caml lineage at INRIA. https://ocaml.org/about
[^2]: Jane Street's use of and investment in OCaml. https://blog.janestreet.com/why-ocaml/
[^3]: OCaml README and 5.0 release notes — multicore runtime, domains, effect handlers, 64-bit-only native compilation from 5.0. https://ocaml.org/releases/5.0.0
[^4]: OCaml README caution note — 4.14 remains supported; maintainers asked to evaluate 5.x and report regressions. https://github.com/ocaml/ocaml/blob/trunk/README.adoc
[^5]: OCaml README, "Separately maintained components" — libraries/tools removed from the core distribution (graphics, num, ocamlbuild, camlp4, Stream/Genlex). https://github.com/ocaml/ocaml/blob/trunk/README.adoc

## Tags

ocaml, functional-programming, compiler, type-inference, ml-family, static-typing, native-code, garbage-collector, effect-handlers, systems-language
