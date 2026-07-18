# racket/typed-racket

> Racket's gradually typed sister language — the longest-running production
> experiment in *sound* gradual typing, with occurrence typing as its core idea.

[GitHub repo](https://github.com/racket/typed-racket) ·
[Documentation](https://docs.racket-lang.org/ts-guide/) ·
[License: Apache-2.0 OR MIT](https://github.com/racket/typed-racket/blob/master/LICENSE)

## Overview

Typed Racket adds statically checked type annotations to Racket. It began as
"Typed Scheme" in Sam Tobin-Hochstadt and Matthias Felleisen's POPL 2008 work[^1]
and was renamed when PLT Scheme became Racket in 2010. Its signature contribution
is **occurrence typing**: the checker narrows union types through ordinary
predicate tests (`(if (string? x) ...)`), so idiomatic untyped Scheme code
typechecks after annotation instead of requiring a rewrite. The design goal was
always migration — port an untyped module, leave the rest alone — and that goal
dictates everything else about the system.

The defining tradeoff is **soundness**. Unlike TypeScript or mypy, Typed Racket
does not erase types: every value crossing a typed/untyped module boundary is
guarded by an automatically generated contract, so a type is a theorem, not a
hint. The cost is real — higher-order wrappers and repeated boundary crossings
produced order-of-magnitude slowdowns in the POPL 2016 evaluation "Is Sound
Gradual Typing Dead?"[^3], the paper that framed a decade of gradual-typing
research. Racket 8.7 (2023) answered with selectable enforcement: `deep`
(classic contracts), `shallow` (transient first-order checks), and `optional`
(erasure, TypeScript-style)[^4].

The repo's 574 stars reflect Racket's niche, not the project's influence: it
ships in every Racket distribution and is among the most-cited artifacts in
programming-language research. Pushed to this week with 348 open issues, it is
actively maintained by the core Racket team at research-project pace.

## Getting Started

Typed Racket is bundled with the [Racket distribution](https://download.racket-lang.org/)[^2];
standalone install:

```bash
raco pkg install typed-racket
```

```racket
#lang typed/racket

;; Occurrence typing: s is (U String False), narrowed by the predicate.
(: len-or-zero (-> (U String False) Index))
(define (len-or-zero s)
  (if (string? s)
      (string-length s)   ; s : String here
      0))

;; Import an untyped library by asserting types — a contract is generated.
(require/typed racket/base
  [string-upcase (-> String String)])
```

## Architecture / How It Works

Typed Racket is not a compiler fork — it is a *library*, the flagship of
Racket's "languages as libraries" architecture[^1]. `#lang typed/racket`
installs a module-level macro that fully expands the module to Racket's small
core language, then runs the type checker over the expanded code at compile
time. This means the checker sees post-macro code: user macros work inside
typed modules without the type system knowing anything about them, but type
errors must be mapped back to source locations through the expansion (the
`source-syntax` collection in this repo does that mapping).

Key subsystems:

- **Occurrence typing** — function types carry logical propositions
  (`string?` returns a proposition about its argument) that the checker
  propagates through `if`, `and`, `or`, and local bindings to refine unions.
  This is the machinery that makes existing Scheme idioms typeable.
- **Numeric tower types** — a large lattice of numeric types
  (`Positive-Byte`, `Index`, `Flonum`, `Exact-Rational`, ...) types Racket's
  numeric tower precisely — and produces baroque error messages.
- **Boundary contracts** — `require/typed` and typed exports compile types to
  contracts. Higher-order functions get chaperone wrappers; mutable vectors
  and structs get impersonators that check every access. This is where
  soundness is paid for.
- **Type-driven optimizer** — after checking, Typed Racket rewrites code to
  specialized unsafe operations (e.g. unboxed float arithmetic), so fully
  typed numeric code can run *faster* than untyped Racket.
- **Enforcement modes** — `typed/racket/shallow` inlines first-order shape
  checks instead of building wrappers (the transient semantics from Greenman's
  deep/shallow work[^5]); `typed/racket/optional` erases checks entirely.

## Production Notes

- **The boundary is the performance model.** Fully typed and fully untyped
  programs are both fast; mixed programs are where the POPL 2016 pathologies
  live[^3]. A tight loop calling an untyped higher-order function through a
  deep boundary can be 10–100x slower. Mitigate by typing whole components,
  keeping hot paths on one side of the boundary, or switching those modules to
  `shallow`.
- **`require/typed` annotations are load-bearing assertions.** You write the
  types for untyped libraries by hand; a wrong annotation surfaces as a
  runtime contract error at best, and in `optional` mode not at all. These
  shims drift as upstream libraries evolve.
- **Not every type crosses.** Contract generation fails for some types (e.g.
  certain polymorphic and mutable combinations); occasionally a value simply
  cannot be exported across a boundary and code must be restructured.
- **Compile times are noticeably worse than untyped Racket** — checking runs
  per module at expansion time. Error messages, especially around the numeric
  tower and polymorphic instantiation, can run to dozens of lines.
- **Occurrence typing does not survive mutation** — narrowing applies to
  immutable bindings; mutable state needs rebinding or `assert`/`cast`.
- **Typed ecosystem coverage is thin.** `typed-racket-more` covers the
  standard library's edges; most third-party packages need hand-written
  `require/typed` shims. Fixes ship with Racket releases, not independently.

## When to Use / When Not

**Use when:**
- You have an untyped Racket codebase to harden incrementally with checked
  (not advisory) guarantees.
- You write numeric/scientific Racket code — the optimizer's float unboxing
  is a genuine speedup over untyped Racket.
- You want sound boundaries with blame, the strongest form of gradual typing
  available in a production language.

**Avoid when:**
- Your program will stay permanently mixed with hot typed/untyped boundaries —
  deep-mode overhead can dominate; measure before committing.
- You depend heavily on third-party Racket packages — budget for writing and
  maintaining `require/typed` shims.
- You are choosing a typed language greenfield — an ML-family language avoids
  the boundary machinery entirely.

## Alternatives

- microsoft/TypeScript — use instead when you want gradual typing with erasure
  semantics on a mainstream ecosystem and accept unsoundness for zero runtime
  cost.
- python/mypy — same erasure tradeoff for Python; annotations as lint, not
  guarantee.
- sorbet/sorbet — gradual typing for Ruby with optional runtime checks; closer
  in spirit to shallow mode.
- coalton-lang/coalton — use instead when you want an ML-style statically
  typed language embedded in a Lisp (Common Lisp) rather than gradual
  migration of existing code.
- racket/racket with `racket/contract` — dynamic contracts without static
  checking, when you want boundary guarantees but not a type system.

## History

| Version | Date | Notes |
|---------|------|-------|
| Typed Scheme | 2008-01 | POPL 2008 paper introduces occurrence typing and the migration design[^1]. |
| Typed Racket | 2010-06 | Renamed in the PLT Scheme → Racket 5.0 rebrand. |
| Standalone repo | 2014-12 | Split into `racket/typed-racket` during Racket's package modularization. |
| — | 2016-01 | "Is Sound Gradual Typing Dead?" (POPL 2016) quantifies boundary overhead across configurations[^3]. |
| Racket 8.7 | 2023-02 | `shallow` and `optional` enforcement modes ship alongside classic `deep`[^4]. |

## References

[^1]: Sam Tobin-Hochstadt, Matthias Felleisen, "The Design and Implementation of Typed Scheme" — POPL 2008. https://doi.org/10.1145/1328438.1328486
[^2]: Typed Racket Guide and Reference. https://docs.racket-lang.org/ts-guide/ · https://docs.racket-lang.org/ts-reference/
[^3]: Asumu Takikawa et al., "Is Sound Gradual Typing Dead?" — POPL 2016. https://doi.org/10.1145/2837614.2837630
[^4]: Racket blog, "Racket v8.7" — 2023-02. https://blog.racket-lang.org/2023/02/racket-v8-7.html
[^5]: Ben Greenman, "Deep and Shallow Types for Gradual Languages" — PLDI 2022. https://doi.org/10.1145/3519939.3523430

## Tags

racket, scheme, lisp, gradual-typing, occurrence-typing, type-checker, contracts, functional-programming, language-tooling
