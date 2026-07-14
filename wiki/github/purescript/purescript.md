# purescript/purescript

> A strongly-typed, Haskell-inspired pure functional language that compiles to JavaScript — with row polymorphism and no runtime, but stuck at 0.15 and slow-moving.

[GitHub repo](https://github.com/purescript/purescript) ·
[Official website](https://www.purescript.org) ·
[License: BSD-3-Clause](https://github.com/purescript/purescript/blob/master/LICENSE)

## Overview

PureScript is a small, strictly-evaluated functional language created by Phil Freeman around 2013[^1]. Syntactically and semantically it is close to Haskell — type classes, higher-kinded types, do-notation, algebraic data types — but it evaluates strictly (not lazily), has no built-in runtime system, and compiles to human-readable JavaScript. The compiler (`purs`) is itself written in Haskell and distributed as a native binary.

Its defining feature is the type system rather than the compile target. PureScript has first-class **row polymorphism**: extensible records and variants typed by open rows, which lets you write functions generic over "any record that at least has these fields." Combined with functional dependencies and a full type-class hierarchy, this makes it one of the more expressive type systems available to front-end developers. The cost is a Haskell-shaped learning curve and a small library ecosystem relative to TypeScript.

The project's central tension is momentum. PureScript never reached 1.0; the current line is 0.15, and releases have slowed to roughly one or two a year[^2]. The language and compiler are stable and the commit stream is active, but many observers describe it as maintenance-paced. Anyone adopting it is betting on a small, devoted community rather than a growing one.

## Getting Started

```bash
# purs (compiler) and spago (build tool) are published as npm wrappers
npm install -g purescript spago
spago init            # scaffold a project
spago run             # compile + run Main.main
```

```purescript
-- src/Main.purs
module Main where

import Prelude
import Effect (Effect)
import Effect.Console (log)

main :: Effect Unit
main = log "Hello, World!"
```

Effects are values: `main` has type `Effect Unit`, and `Effect` is a first-class type describing a synchronous side-effecting computation, not an implicit ambient capability.

## Architecture / How It Works

The compiler front-end parses `.purs` modules, performs type inference (bidirectional, with type classes resolved by instance search), and lowers to an intermediate representation called **CoreFn**. The default back-end then emits JavaScript from CoreFn. Because CoreFn is a documented, serializable IR (emit it with `purs compile --dump-corefn`), alternative back-ends exist that reuse the entire front-end and swap only code generation: `purerl` targets Erlang, and there are community back-ends for C++, Go, and others. This is the cleanest part of the design — the type checker is decoupled from the target.

The JavaScript output is deliberately readable and maps closely to the source. There is no runtime library injected: PureScript values are plain JS values (records are objects, functions are curried JS functions), which is what makes the FFI cheap. Foreign functions are written in a sibling `.js` file and imported via `foreign import`, with the PureScript side supplying only a type signature. This tight, unmanaged coupling to JavaScript is a strength (interop is direct) and a footgun (a wrong FFI type signature is unchecked and fails at runtime).

Since 0.15 (2022) the back-end emits **ES modules** and the older CommonJS output was dropped[^3]. Earlier the 0.14 line (2021) rewrote the kind system to support polymorphic kinds, unifying types and kinds[^4]. These were the two most disruptive migrations in recent memory; each broke library code across the ecosystem.

Build and dependency management live outside the compiler. The modern tool is **Spago**, which resolves dependencies against curated *package sets* (a pinned, mutually-compatible snapshot of the ecosystem) rather than solving version ranges per-package. Spago itself was rewritten from Haskell to PureScript, and its config format moved from Dhall to YAML in that transition.

## Production Notes

- **The ecosystem is package-set-shaped.** You do not pick arbitrary library versions; you pick a package set and get whatever versions it pins. This makes builds reproducible and avoids version-solving hell, but adding a library that isn't in your package set, or upgrading one ahead of the set, is manual work.
- **Compiler upgrades are ecosystem events.** Because `core` libraries track compiler releases, moving from one 0.1x line to the next (especially 0.13→0.14→0.15) typically means waiting for the package set to catch up and editing FFI files. Do not upgrade the compiler independently of the package set.
- **FFI is unchecked.** The boundary to JavaScript is only as correct as the type signatures you hand-write over foreign code. Bad signatures compile cleanly and blow up at runtime; wrap third-party JS carefully.
- **Bundle size and dead-code elimination.** Output is one JS module per PureScript module; you rely on a downstream bundler (esbuild is common) plus `purs bundle`/tree-shaking to drop unused code. Naive builds ship more than you expect.
- **Tooling is thinner than TypeScript's.** `purs ide` powers editor integration (autocomplete, go-to-definition) and generally works, but the IDE server, formatter (`purs-tidy`), and language-server layers are community-maintained and less polished than the TS toolchain. Expect occasional friction.
- **Hiring and documentation.** The talent pool is small and much learning material predates the ES-modules/kinds changes, so older tutorials contain code that no longer compiles. The book[^5] and Pursuit[^6] (the typed package-doc index) are the reliable references.

## When to Use / When Not

**Use when:**
- You want Haskell-grade types (type classes, HKTs, row polymorphism) on the front end and are willing to pay the learning cost.
- You value a pure, effect-as-value model and readable JS output with a thin, direct FFI.
- Your team already knows Haskell/functional programming and treats the small ecosystem as acceptable.

**Avoid when:**
- You need a large library ecosystem, abundant hiring, and mainstream tooling — TypeScript wins decisively there.
- You want gradual adoption inside an existing JS/TS codebase; PureScript is an all-in language boundary, not a superset.
- You need assurance of fast-moving upstream development or a 1.0 stability guarantee.

## Alternatives

- microsoft/typescript — the pragmatic default; structural types and a huge ecosystem, but far weaker guarantees than PureScript's sound type system. Use instead when interop and hiring matter more than type strength.
- ghcjs/ghcjs (and GHC's JS backend) — real Haskell to JavaScript with laziness and the Haskell ecosystem, at the cost of larger output and heavier runtime. Use when you want actual Haskell, not a JS-native FP language.
- elm/compiler — smaller, opinionated pure-FP language for front-end apps with famously friendly errors and no type classes. Use when you want guardrails and simplicity over expressiveness.
- rescript-lang/rescript — OCaml-derived, fast compiler, strong React interop, more JS-pragmatic than PureScript. Use when you want ML-family types with first-class JS/React ergonomics.
- gleam-lang/gleam — statically-typed FP compiling to Erlang and JavaScript; simpler type system, growing community. Use when actor-model/BEAM targets or momentum matter.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013 | Initial public release by Phil Freeman[^1]. |
| 0.12 | 2018 | Compiler and core-library modernization. |
| 0.13 | 2019 | Continued type-system and tooling improvements. |
| 0.14 | 2021 | Polymorphic kinds; unified type/kind system[^4]. |
| 0.15 | 2022-05 | ES-modules output; CommonJS backend dropped[^3]. |
| 0.15.16 | 2026-03-15 | Latest release as of writing[^2]. |

## References

[^1]: PureScript — official site and history. https://www.purescript.org
[^2]: PureScript releases & changelog. https://github.com/purescript/purescript/releases
[^3]: PureScript 0.15.0 release notes — ES modules backend, CommonJS removed. https://github.com/purescript/purescript/releases/tag/v0.15.0
[^4]: PureScript 0.14.0 release notes — polymorphic kinds. https://github.com/purescript/purescript/releases/tag/v0.14.0
[^5]: "PureScript by Example" (the PureScript book). https://book.purescript.org/
[^6]: Pursuit — PureScript package and documentation index. https://pursuit.purescript.org/

## Tags

purescript, functional-programming, compile-to-javascript, haskell, strongly-typed, type-classes, row-polymorphism, alt-js, frontend, spago
