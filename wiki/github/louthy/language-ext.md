# louthy/language-ext

> A functional-programming framework for C# that recreates Haskell/ML idioms — higher-kinded types, monads, immutable collections, effect systems — inside the BCL.

[GitHub repo](https://github.com/louthy/language-ext) ·
[API Reference](https://louthy.github.io/language-ext/) ·
[License: MIT](https://github.com/louthy/language-ext/blob/main/LICENSE.md)

## Overview

language-ext is Paul Louth's long-running attempt to make C# feel like a pure functional language[^1]. Started in 2014, it predates most of C#'s functional additions (records, pattern matching, nullable reference types) and has grown into a large framework: optional/either monads, immutable collections, software transactional memory, parser combinators, an effect/IO system, and functional streaming. The stated goal is to bias an engineer's "inertia" toward declarative, pure, side-effect-controlled code rather than imperative mutation[^1].

The defining tension is that C# has no native higher-kinded types (you cannot write `M<A>` where `M` is a type parameter), so the library simulates them. Since the v5 rewrite this is done through a trait encoding — `K<M, A>` plus trait interfaces like `Monad<M>`, `Applicative<F>`, `Traversable<T>` — that lets generic code abstract over "any monad"[^2]. This is genuinely expressive but non-idiomatic: it produces deep generic signatures, camelCase `Prelude` functions that look nothing like typical C#, and compiler errors that can be hard to read. It is a framework you adopt as a paradigm, not a utility library you sprinkle in.

The other defining trait is churn. Major versions are not gentle refactors — v5 (2024–2025) was a near-total rewrite of the type-class machinery and effect system, and migrating a v4 codebase is a real project, not a package bump[^2].

## Getting Started

```bash
dotnet add package LanguageExt.Core
```

Set up `global using` so the `Prelude` constructor functions are in scope everywhere:

```csharp
global using LanguageExt;
global using static LanguageExt.Prelude;
```

```csharp
// Option instead of null; Either/Fin instead of exceptions
Option<int> Parse(string s) =>
    int.TryParse(s, out var n) ? Some(n) : None;

// LINQ query syntax works as do-notation for any monad
var result =
    from a in Parse("10")
    from b in Parse("32")
    select a + b;                 // Option<int> = Some(42)

int answer = result.IfNone(0);    // 42
```

## Architecture / How It Works

The framework is split across several NuGet packages so you only pull in what you use:

- **LanguageExt.Core** — the bulk: `Option`, `Either`, `Fin`, `Validation`, `Try`, immutable collections, `Atom`/`Ref` STM, the trait/type-class system, and the `IO`/`Eff` effect monads.
- **LanguageExt.Parsec** — a port of Haskell's `parsec` parser-combinator library[^3].
- **LanguageExt.Streaming** — compositional streaming types (`Source`, `Sink`, `Conduit`, `Pipes`).
- **LanguageExt.FSharp** — interop between the Core types and F#'s `Option`/`List`/`Map`.
- **LanguageExt.Rx** — Reactive Extensions bridges for Core types.
- **LanguageExt.Sys** — a pure, unit-testable wrapper over `System` IO (file, console, time) driven through an injectable runtime.

**Higher-kinded traits.** Because C# cannot express `M<_>` as a parameter, v5 introduces the surrogate `K<M, A>` (read: "some `M` applied to `A`"). Traits like `Monad<M>`, `Functor<F>`, and `Foldable<T>` are static-abstract interfaces implemented by each type. Generic algorithms (`traverse`, `sequence`, `bind`) are written once against the trait and reused across every concrete monad[^2].

**Effects.** `IO<A>` is the base side-effecting monad; `Eff<A>` adds error handling via the `Error` type; `Eff<RT, A>` adds an injectable runtime `RT` so effects (time, file system, environment) become dependency-injected and testable without mocks. Both are lazy — nothing runs until you `.Run()`.

**Immutable collections.** `Seq<A>` (lazy, evaluate-at-most-once list), `Lst<A>`, `Arr<A>`, `Map<K,V>`, `HashMap<K,V>`, `Set<A>`, `HashSet<A>`, `Que<A>`, `Stck<A>`. Ord/Eq-constrained variants (`Map<OrdK,K,V>`) let you pin comparison behavior at the type level.

**Concurrency.** `Atom<A>` is a lock-free atomically-updatable reference; `Ref<A>` participates in an STM (`atomic(...)`) transaction system modeled on Clojure's. `AtomHashMap` and vector-clock types support shared and distributed state.

Query syntax (`from … select`) is repurposed as do-notation: LINQ's `SelectMany` becomes monadic bind, so the same `from/in/select` block composes `Option`, `Either`, `Eff`, or any user monad.

## Production Notes

- **Compile times and error messages.** The trait-based HKT encoding generates deep, nested generic types. Large codebases can see noticeably slower builds and IntelliSense, and a single mismatch can produce multi-line generic errors that don't point cleanly at the mistake. This is the most common real-world complaint.
- **Migration cost is real.** v5 reworked the type classes, effect system, and much of the surface area; there is no mechanical v4→v5 upgrade. Pin your major version and budget time before upgrading. Treat the major-version boundary as a rewrite, not a patch[^2].
- **Non-idiomatic by design.** camelCase `Prelude` functions, constructor functions instead of `new`, and pervasive `global using` clash with standard C# conventions and with teammates/linters expecting BCL style. Every camelCase function has a PascalCase fluent equivalent if you want to hide the ML flavor.
- **Async bridging.** `IO`/`Eff` are lazy and have their own run/await model; interleaving them with existing `Task`-based code (ASP.NET pipelines, EF Core) requires care at the boundary and is a frequent source of confusion.
- **Allocation.** Immutable collections and monadic wrapping allocate more than mutable BCL equivalents. `Seq` is heavily optimized, but hot inner loops that were `List<T>`/`Dictionary<K,V>` mutation can regress if naively ported.
- **Team buy-in is a prerequisite.** The library only pays off if the whole team writes in its idiom; a mixed codebase gets the costs (unfamiliar types, learning curve) without the consistency benefit.

## When to Use / When Not

**Use when:**
- You want Haskell/F#-style total functions, no-null, and explicit error handling but are committed to staying on C#/.NET for tooling, hiring, or ecosystem reasons.
- You're building domain logic where `Option`/`Either`/`Validation` and immutability materially reduce bugs.
- The whole team is willing to adopt functional idioms and the effect/runtime testing model.

**Avoid when:**
- You want a small, idiomatic helper library — the framework is all-in, not à la carte in spirit.
- Your team is unfamiliar with FP and can't absorb the learning curve and non-standard style.
- You're in allocation-sensitive hot paths, or need maximal build speed and clean IntelliSense.
- You could just use F#, where higher-kinded abstraction and immutability are native rather than simulated.

## Alternatives

- louthy/language-ext vs the language — if you can pick the language, dotnet/fsharp gives you native immutability, discriminated unions, and computation expressions without an HKT surrogate.
- nlkl/Optional — use when you only want a small, idiomatic `Option<T>` type and nothing else.
- louthy/csharp-monad — Paul Louth's older, smaller monad library; superseded by language-ext but lighter.
- Simon-Hutton/OneOf (louthy alternative) — use OneOf/OneOf when you specifically want discriminated-union return types without adopting a whole FP framework.
- vkhorikov/CSharpFunctionalExtensions — use when you want `Result`/`Maybe` and railway-oriented programming in a deliberately conservative, BCL-friendly style.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2014-11 | Project created by Paul Louth[^1]. |
| 1.x | ~2016 | Core `Option`/`Either`/immutable collections established. |
| 3.x | ~2018 | Broadened trait/type-class experiments, Parsec, code-gen. |
| 4.0 | ~2021 | Major line: nullable-reference-type support, `Eff`/`Aff` effect monads, .NET Core focus. |
| 5.0 | ~2024–2025 | Rewrite: `K<M,A>` trait-based higher-kinded types, unified `IO`/`Eff`, `LanguageExt.Streaming`[^2]. |
| main | 2026-06 | Active; last push 2026-06-18, ~7.1k stars, MIT. |

## References

[^1]: language-ext README and project — "C# Functional Programming Language Extensions." https://github.com/louthy/language-ext
[^2]: language-ext API reference (v5 traits, effects, streaming). https://louthy.github.io/language-ext/
[^3]: LanguageExt.Parsec — port of the Haskell parsec library. https://www.nuget.org/packages/LanguageExt.Parsec

## Tags

csharp, dotnet, functional-programming, monad, higher-kinded-types, immutable-collections, effect-system, parser-combinators, software-transactional-memory, fsharp, library
