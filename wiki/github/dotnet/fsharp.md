# dotnet/fsharp

> The F# compiler, FSharp.Core, and editor tooling — the ML-family functional-first language that lives inside a C#-first platform.

[GitHub repo](https://github.com/dotnet/fsharp) ·
[Official website](https://dotnet.microsoft.com/languages/fsharp) ·
[License: MIT](https://github.com/dotnet/fsharp/blob/main/License.txt)

## Overview

F# is the functional-first language of .NET: an ML-family language (Hindley–Milner-style type inference, immutability by default, algebraic data types, pattern matching) with full access to the .NET runtime and library ecosystem. It was created by Don Syme at Microsoft Research, first released in 2005, and became a supported Visual Studio language with F# 2.0 in 2010[^1]. This repo is the monorepo for the compiler, the FSharp.Core standard library, FSharp.Compiler.Service (the compiler-as-a-library that powers every F# editor), and the Visual Studio integration.

The star count (~4.3k) wildly understates usage: almost nobody installs F# from this repo. The compiler ships inside the .NET SDK, so `dotnet` users get it for free, and the repo functions as a development venue rather than a distribution channel. Language design deliberately does not happen here either — suggestions go to fslang-suggestions, approved ideas become RFCs in fslang-design, and only implementations land in this repo[^2].

The defining tension of F# is platform citizenship. Being a guest on a platform whose primary language is C# buys F# a world-class runtime, GC, libraries, and deployment story that OCaml and Haskell users assemble themselves — at the cost of forever tracking C#-driven runtime features (nullability annotations, `Span<T>`, static abstract interface members) and accepting that tooling investment prioritizes Visual Studio and the Microsoft release cadence. Releases are versioned in lockstep with .NET: a new major F# ships each November with the annual .NET release.

## Getting Started

Install the [.NET SDK](https://dotnet.microsoft.com/download); the F# compiler is included. Then:

```bash
dotnet new console -lang F# -o MyApp
cd MyApp
dotnet run
```

```fsharp
// Program.fs — discriminated unions + exhaustive pattern matching
type Shape =
    | Circle of radius: float
    | Rect of w: float * h: float

let area shape =
    match shape with
    | Circle r -> System.Math.PI * r * r
    | Rect (w, h) -> w * h

[ Circle 1.0; Rect (2.0, 3.0) ]
|> List.map area
|> List.iter (printfn "%f")
```

For exploratory work, `dotnet fsi` starts the F# Interactive REPL, which can reference NuGet packages inline (`#r "nuget: Newtonsoft.Json"`).

## Architecture / How It Works

The compiler is itself written in F# and follows a classical pipeline[^3]: lexing and parsing (via the repo's own fslex/fsyacc tools) into an untyped syntax tree, then type checking into a typed tree, then optimization, then IL generation ("AbsIL") producing standard .NET assemblies. The type checker is the heart and the beast: `CheckExpressions.fs` is one of the largest single source files you will find in a production compiler, and it is where ML-style inference, overload resolution, and .NET interop rules collide.

Two structural decisions shape everything downstream:

- **File order matters.** F# compiles files in project order, single pass, and forward references across files are disallowed. This is a deliberate design choice (it makes dependency structure explicit and code readable top-to-bottom) that doubles as the language's most-argued-about property. It also historically serialized type checking; graph-based parallel type checking landed as an opt-in in F# 8[^4].
- **The compiler is a library.** FSharp.Compiler.Service (FCS) is the same codebase packaged as a NuGet library with incremental checking. Ionide (VS Code), Rider, and the Visual Studio tooling in this repo all sit on FCS. This is why editor behavior is consistent across IDEs — and why a type-checker performance bug degrades every editor at once.

The repo also carries `vsintegration/`, the Visual Studio F# tools, and its branch structure reflects that coupling: `main` targets the latest public VS, while `release/dev17.x` branches track specific Visual Studio point releases. FSharp.Core is versioned independently, and FSharp.Compiler.Service uses its own 43.x scheme — a historical artifact that confuses newcomers.

## Production Notes

- **FSharp.Core version pinning.** Every F# assembly references FSharp.Core, and mixed versions across NuGet dependencies cause binder headaches. Pin an explicit `FSharp.Core` PackageReference at the application level; do not rely on transitive resolution.
- **Compile times.** F# type checking is slower than C#'s, and large projects with long file chains feel it. Mitigations: split into more, smaller projects; enable graph-based checking (opt-in since F# 8[^4]); keep signature files (`.fsi`) for large modules, which also speeds incremental editor checking.
- **`async` vs `task`.** F#'s original `async { }` computation expression predates .NET's `Task` and does not interoperate with it for free. Since F# 6, `task { }` compiles to the same state machines C# generates and is the right default for .NET-facing and performance-sensitive code[^5]; `async` remains idiomatic for F#-only compositional workflows. Teams mixing both without a policy accumulate adapter noise (`Async.AwaitTask`).
- **C# consuming F# is the rough direction.** F# consumes C# libraries cleanly; the reverse is awkward — discriminated unions surface as opaque class hierarchies, `Option<'T>` and curried functions leak compiler-generated shapes. If you ship a library for C# consumers, design a dedicated C#-friendly facade.
- **Nullability arrived late.** Nullable reference type support (interop with C#'s `string?` annotations) only landed in F# 9 (2024)[^6]; before that, F# code treated C# nullability annotations as invisible. Expect a migration tail in older codebases.
- **Tooling outside Visual Studio is community-run.** Ionide and fsautocomplete are community projects on top of FCS. Quality is good but release timing can lag SDK releases; after each November .NET drop, expect a few weeks of editor-plugin churn.
- **Repo health.** Actively developed: pushed within the last day as of this writing, ~1.2k open issues+PRs, with Microsoft staff and long-time community contributors both landing changes. The "help wanted" / "good first issue" funnel is genuinely maintained.

## When to Use / When Not

**Use when:**
- You want ML-family typed functional programming with a production runtime, mainstream deployment story, and access to the entire NuGet ecosystem.
- Your domain rewards algebraic modeling: finance, trading, insurance, compilers/DSLs, data transformation pipelines.
- You are already on .NET and want lower-defect domain code without leaving the platform (mixed C#/F# solutions are routine).
- You want scripting plus types: `dotnet fsi` with inline NuGet references is a strong glue-code story.

**Avoid when:**
- Your team ships libraries primarily for C# consumers and cannot afford facade maintenance — the interop asymmetry is a real tax.
- You need maximum hiring pool and framework documentation coverage on .NET — C# is the paved road, and most ASP.NET Core docs assume it.
- You want a functional language with no corporate platform coupling — F#'s roadmap follows the .NET release train.
- Your workload is systems-level with no GC tolerance; F# is a managed language, full stop.

## Alternatives

- dotnet/roslyn — use C# instead when team familiarity and first-party framework docs outweigh F#'s modeling advantages on the same runtime.
- ocaml/ocaml — use OCaml instead when you want the ML core without .NET, with faster native compilation and no platform-vendor coupling.
- ghc/ghc — use Haskell instead when you want pure, lazy functional programming and maximal type-system expressiveness over runtime pragmatism.
- scala/scala — use Scala instead when your ecosystem is the JVM (Spark, Kafka, Akka) rather than .NET.
- gleam-lang/gleam — use Gleam instead when you want a small ML-flavored typed language targeting the BEAM/Erlang runtime for concurrency-heavy services.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2005 | First release from Microsoft Research[^1]. |
| 2.0 | 2010-04 | Shipped in Visual Studio 2010; first fully supported product release. |
| 3.0 | 2012 | Type providers, query expressions. |
| 4.0 | 2015-07 | Open development on GitHub (this repo, created 2015-01) as microsoft/visualfsharp. |
| 4.1 | 2017 | .NET Core support, struct tuples. |
| — | 2019 | Repo renamed to dotnet/fsharp. |
| 5.0 | 2020-11 | .NET 5 alignment; string interpolation, `nameof`. |
| 6.0 | 2021-11 | `task { }` computation expression, uniform indexing syntax[^5]. |
| 7.0 | 2022-11 | Static abstract interface member interop, simplified generic syntax. |
| 8.0 | 2023-11 | Shorthand lambdas (`_.Prop`), nested record updates, opt-in graph-based type checking[^4]. |
| 9.0 | 2024-11 | Nullable reference types, auto `.Is*` properties on unions[^6]. |
| 10 | 2025-11 | Shipped with .NET 10. |

## References

[^1]: Don Syme, "The Early History of F#" — HOPL IV, 2020. https://fsharp.org/history/hopl-final/hopl-fsharp.pdf
[^2]: F# Language Design Process (suggestions → RFCs → implementation). https://github.com/fsharp/fslang-design
[^3]: The F# Compiler Guide — architecture docs for this repo. https://fsharp.github.io/fsharp-compiler-docs/
[^4]: .NET Blog, "Announcing F# 8" — 2023-11. https://devblogs.microsoft.com/dotnet/announcing-fsharp-8/
[^5]: .NET Blog, "What's new in F# 6" — 2021-11. https://devblogs.microsoft.com/dotnet/whats-new-in-fsharp-6/
[^6]: .NET Blog, "Announcing F# 9" — 2024-11. https://devblogs.microsoft.com/dotnet/announcing-fsharp-9/

## Tags

fsharp, dotnet, compiler, functional-programming, programming-language, ml-family, type-inference, language-tooling, visual-studio
