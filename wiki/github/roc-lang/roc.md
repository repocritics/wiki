# roc-lang/roc

> A pure functional language that aims for ML-family ergonomics and C-adjacent speed — built around a platform/application split, and still pre-0.1 after years of development.

[GitHub repo](https://github.com/roc-lang/roc) ·
[Official website](https://roc-lang.org) ·
[License: UPL-1.0](https://github.com/roc-lang/roc/blob/main/LICENSE)

## Overview

Roc is a purely functional programming language created by Richard Feldman, known previously for his work in the Elm community[^1]. It inherits Elm's design sensibilities — no runtime exceptions, whole-program type inference with no mandatory annotations, friendly compiler messages — but targets native code (and WebAssembly) rather than the browser, and is deliberately positioned as a general-purpose language rather than a frontend one[^2].

The defining architectural idea is the **platform/application split**[^3]. An application contains pure business logic and declares which *platform* it builds against. The platform — written in a systems language and authored separately — provides the host, the entry point, and all effectful primitives (I/O, networking, time). The same application logic can, in principle, run on a CLI platform, a web-server platform, or an embedded platform by swapping the platform declaration. This is Roc's biggest bet and its central tension: it buys genuine portability and a clean purity boundary, but fragments the ecosystem into per-platform silos and pushes a lot of complexity onto the small population of platform authors.

The other defining fact is maturity. The repository was created in 2019[^gh] and, as of 2026, Roc is still not at a 0.1 release — the README states this plainly[^2]. Development is nevertheless active (commits land essentially daily), and the compiler is in the middle of a ground-up rewrite: originally written in Rust, the current in-tree compiler is written in **Zig**[^gh]. Anyone evaluating Roc should treat syntax, the standard library, and platform APIs as unstable and subject to breaking change without notice.

## Getting Started

There is no stable release. Install a nightly build from the official install guide[^4]:

```bash
# download a nightly from https://roc-lang.org/install, then:
roc version
roc dev main.roc     # type-check + run
roc run main.roc     # run
roc build main.roc   # produce a native binary
```

A minimal application targeting a CLI platform (syntax is pre-0.1 and changes often — check the current tutorial):

```roc
app [main!] { pf: platform "https://github.com/roc-lang/basic-cli/releases/.../platform/main.roc" }

import pf.Stdout

main! = \_args ->
    Stdout.line! "Hello, World!"
```

The `!` suffix marks effectful calls, reflecting Roc's move to make effects visible in the syntax rather than tracked purely in types[^5].

## Architecture / How It Works

**Compilation.** Roc source is parsed, type-inferred, monomorphized, and lowered to machine code. The long-standing backend used LLVM for optimized builds, with a faster development backend for quick `roc dev` iteration[^3]. The compiler is being rewritten in Zig, so internal stages and IRs are in flux; the "new compiler" tutorial in the repo reflects this transition[^2].

**Type system.** Whole-program Hindley–Milner-style inference means annotations are almost always optional. Data is modeled with records, *tag unions* (structural sum types that unify by shape), and opaque types. Roc has **abilities** — its answer to type classes / traits — for constrained polymorphism such as equality, hashing, and encoding[^3].

**Memory management.** Roc has no garbage collector and no manual `free`. It uses automatic **reference counting** with opportunistic in-place mutation: when the compiler can prove a value is uniquely referenced, it mutates in place instead of copying, which lets idiomatic pure code approach the performance of imperative mutation[^6]. This lineage is shared with Koka's Perceus reference counting.

**Platforms.** A platform is the trust-and-effects boundary. It is written in a host language (Rust, Zig, C, or anything that can produce the required object interface), defines the program's entry point, and exposes an effect API the application calls into. Applications cannot perform I/O directly — every effect is a function the platform provided. `basic-cli` and `basic-webserver` are the reference platforms most newcomers use[^3].

## Production Notes

Roc is **not production-ready and does not claim to be**. The most important operator caveat is that there are no version guarantees: no 0.1, no semver, no backward-compatibility promise. Code written today may not compile on next month's nightly, and platform releases are pinned by URL to specific compiler builds.

- **Toolchain pinning is mandatory.** Because compiler, standard library, and platforms co-evolve, a working project is a specific `(nightly, platform-release)` pair. Upgrading the compiler generally means upgrading platforms in lockstep; mismatches surface as opaque link/ABI errors, not friendly diagnostics.
- **The compiler rewrite is a moving target.** With the Zig rewrite underway, some features present in older Rust-compiler builds may be temporarily absent or behave differently in current builds. Tutorials and blog posts written against the older compiler can be stale on syntax and available builtins.
- **Small ecosystem, few platforms.** Practical use is bounded by which platforms exist. Outside `basic-cli` / `basic-webserver` and a handful of community platforms, you will be writing your own host — which requires systems-language skill and an understanding of Roc's calling convention and refcount ABI.
- **Debugging across the boundary.** Panics and effect errors originate in the platform host; stack traces cross the Roc/host boundary and are less legible than a single-language stack. Tooling (LSP, formatter, debugger integration) exists but is early.
- **No package registry.** Dependencies are fetched by URL to release artifacts rather than resolved through a central registry; there is no `cargo`/`npm`-equivalent lockfile-and-index workflow yet.

Interpreting the signals: ~5.8k stars and daily commits over seven years indicate a serious, sustained effort with a committed core — but the persistent pre-0.1 status is the honest headline. Roc is a language to learn, prototype with, and follow, not one to ship a business on in 2026.

## When to Use / When Not

**Use when:**
- You want to learn a modern pure-functional language with excellent error messages and near-imperative performance.
- You're exploring the platform/application model or writing a host to embed Roc logic in a larger system.
- You're building throwaway CLIs, scripts, or experiments and can tolerate churn.

**Avoid when:**
- You need stability, backward compatibility, or a hiring pool — none exist yet.
- You need a mature package ecosystem or third-party library depth.
- You're shipping production services or anything with a multi-year maintenance horizon.
- You need a language spec or long-term-support guarantees for compliance reasons.

## Alternatives

- gleam-lang/gleam — friendly, type-safe functional language on the BEAM/JS; use Gleam when you want a *released, stable* friendly-FP language with real concurrency today.
- elm/compiler — same creator lineage and design DNA, but browser-only; use Elm when your target is a reliable frontend SPA.
- koka-lang/koka — research language sharing Roc's Perceus-style reference counting plus first-class effect handlers; use Koka to study effect typing.
- ocaml/ocaml — mature ML-family language with native compilation and a real ecosystem; use OCaml when you need functional + fast + production-ready now.
- purescript/purescript — pure, Haskell-like functional language compiling to JS; use PureScript when you want Haskell semantics on the web.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Repo created | 2019-01-20 | Development begins; original compiler in Rust[^gh]. |
| Public talks | 2021– | Richard Feldman begins presenting Roc publicly[^1]. |
| Zig compiler rewrite | ~2024–2026 | Compiler rewritten in Zig; repo primary language now Zig (rewrite dates approximate)[^gh]. |
| Still pre-0.1 | 2026-07 | No tagged release; active daily development; last push 2026-07-14[^gh][^2]. |

## References

[^1]: Richard Feldman — creator of Roc, formerly prominent in the Elm community. https://github.com/rtfeldman
[^2]: Roc README — "Work in progress! Roc is not ready for a 0.1 release yet." https://github.com/roc-lang/roc
[^3]: Roc official site — language overview, platforms, and abilities. https://roc-lang.org
[^4]: Roc install guide (nightly builds). https://roc-lang.org/install
[^5]: Roc FAQ — effects and purity. https://roc-lang.org/faq
[^6]: Roc FAQ — memory management via opportunistic in-place mutation and reference counting. https://roc-lang.org/faq
[^gh]: GitHub API, repos/roc-lang/roc — created 2019-01-20, primary language Zig, license UPL-1.0, last push 2026-07-14, ~5,778 stars (fetched 2026-07-15).

## Tags

functional-programming, pure-functional, programming-language, compiler, zig, native-code, type-inference, platform-application, reference-counting, pre-release
