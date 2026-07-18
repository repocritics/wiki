# fable-compiler/Fable

> The F# to JavaScript (and TypeScript, Python, Rust, Dart, Erlang) compiler —
> F# as a first-class citizen of other language ecosystems.

[GitHub repo](https://github.com/fable-compiler/Fable) ·
[Official website](https://fable.io) ·
[License: MIT](https://github.com/fable-compiler/Fable/blob/main/LICENSE)

## Overview

Fable compiles F# source to idiomatic JavaScript, letting .NET developers write
browser frontends without leaving the language[^1]. Started in early 2016 by
Alfonso García-Caro, the name is a nod to Babel: early Fable emitted a Babel
AST ("F# |> BABEL" was the original tagline). Since Fable 3 (2020) it is a
self-contained `dotnet` tool that emits plain ES-module `.fs.js` files consumed
by any bundler with no Babel dependency[^2]. Fable 4 widened the story from
"F# to JS" to "F# to X": TypeScript, Python, Rust, and Dart backends (the
non-JS ones officially beta), with an Erlang/Beam target as the most recent
addition per the repo's own description.

The defining tension is semantic fidelity vs. output cost. Fable does not ship
a .NET runtime to the browser (that is Blazor's trade); it reimplements F#
core semantics — immutable lists, Map/Set, int64, decimal, structural
equality, string formatting — as a per-target `fable-library`. You get
readable, tree-shakeable output, but you inherit an emulation tax wherever F#
semantics diverge from the host platform's.

At ~3.1k stars and 329 forks it is a niche-language tool, not a mass-market
one — but it is the center of gravity of the F# frontend ecosystem (Elmish,
Feliz, the SAFE stack[^3]) and actively maintained, with pushes landing in
July 2026 and a moderate 165 open issues.

## Getting Started

```bash
dotnet new tool-manifest          # once per repo
dotnet tool install fable
dotnet fable src/App.fsproj --outDir dist    # or: dotnet fable watch
```

```fsharp
// App.fs — needs the Fable.Browser.Dom NuGet package
module App

open Browser.Dom

let mutable count = 0
let button = document.querySelector "#inc" :?> Browser.Types.HTMLButtonElement

button.onclick <- fun _ ->
    count <- count + 1
    document.getElementById("count").textContent <- string count
```

Fable only produces JS files; a bundler/dev server (Vite is the common
pairing) serves and packages them. Most real apps start from an Elmish or
SAFE stack template rather than raw DOM calls[^3].

## Architecture / How It Works

The frontend is **F# Compiler Services (FCS)** — but not stock FCS. Fable uses
a patched fork (`ncave/fsharp`) whose binaries are vendored in `lib/fcs`[^4].
This lets the compiler run in unusual hosts (see fable-standalone below) at
the cost of a second F# compiler in the ecosystem: your IDE type-checks with
real FCS, Fable compiles with the fork, and the two occasionally disagree on
bleeding-edge language features until the fork catches up.

The pipeline: FCS parses and type-checks the `.fsproj` → the typed AST is
transformed into **Fable AST** (published as the `Fable.AST` package) → a
per-language backend applies target-specific transforms and prints source.
Python, Rust, and Dart each get their own printer plus their own port of
`fable-library` (`-py`, `-rust`, `-dart`), because "what is an F# list" has a
different answer in each target. The Rust backend is the most ambitious,
mapping GC semantics onto reference counting.

Interop lives in **Fable.Core**: `[<Import>]` and `[<Emit>]` attributes, the
dynamic `?` operator, `[<Erase>]` unions and `[<StringEnum>]` for zero-cost
typed wrappers over untyped JS. **fable-standalone** is the compiler compiled
to JS by itself, powering the browser-only REPL[^5]. Compiler plugins (e.g.
Feliz's JSX-style transform) operate on Fable AST and are binary-coupled to
the `Fable.AST` version — a recurring break point across major releases.

## Production Notes

- **The emulation tax is real but bounded.** F# lists, Map, Set, int64,
  decimal, and `sprintf`-style formatting are library implementations, not
  native JS. int64/decimal arithmetic in hot loops is markedly slower than
  `number` math; prefer native ints/floats and `ResizeArray` (a JS array)
  in performance-critical paths.
- **Structural equality is a function call.** `=` on records/unions compiles
  to a deep-equality helper, not `===` — a surprise when Fable values end up
  as React deps or keys in identity-keyed JS collections.
- **BCL coverage is partial.** Reflection is limited, regex is the host's
  engine with different semantics from .NET's, DateTime edge cases differ.
  Check the compatibility docs before porting server-side F# wholesale[^6].
- **Bindings are the interop bottleneck.** Popular libraries have maintained
  bindings (Fable.React, Feliz); the long tail means writing your own or
  running `ts2fable` over `.d.ts` files, whose output needs hand repair[^7].
- **Upgrade pains follow major versions.** Fable 2→3 discarded the entire
  Babel/webpack-loader plugin ecosystem for the dotnet-tool model; Fable 3→4
  required plugin authors to recompile against the new Fable.AST. Pin the
  tool version in the manifest and upgrade deliberately.
- **Treat non-JS targets as beta, because they are.** JS/TS is the production
  path; Python, Rust, Dart have known gaps, and Erlang/Beam is newer still.
- **Cold compiles pay .NET startup plus type-checking.** `dotnet fable watch`
  is the workable inner loop, but large solutions feel slower than
  esbuild-class JS tooling.

## When to Use / When Not

**Use when:**
- Your team already writes F# and wants to share domain types and logic
  between a .NET backend and a browser frontend (Fable.Remoting, SAFE stack).
- You want Elmish-style MVU with a mature typed-FP language and .NET tooling.
- You want readable JS that plugs into a standard bundler, not a runtime.

**Avoid when:**
- The team has no F# background and just wants typed JS — TypeScript or
  ReScript reach the same goal without a second toolchain.
- You need full .NET runtime semantics in the browser — Blazor/Bolero ship
  the actual runtime instead of emulating it.
- The app is mostly thin glue over many JS libraries — the binding tax
  dominates the benefit.
- You were hoping to ship the Python/Rust/Dart targets to production today.

## Alternatives

- fsbolero/Bolero — use when you want the real .NET runtime in the browser
  (Blazor WebAssembly) instead of transpiled JS.
- dotnet-websharper/core — older full-stack F#/C# web framework with its own
  JS translation; heavier, more framework-shaped.
- rescript-lang/rescript-compiler — typed functional-to-JS without the .NET
  dependency; leaner toolchain, no BCL emulation.
- elm/compiler — the MVU language Elmish borrowed from; simpler, more
  locked-down, no .NET interop.
- microsoft/TypeScript — stay inside the JS ecosystem when gradual typing is
  enough and language-level FP is not the point.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2016-01 | Project started; compiled F# to a Babel AST. |
| 1.0 | 2017 | First stable line; Babel-based toolchain. |
| 2.0 | 2018-10 | Rewrite focused on smaller, more idiomatic JS output. |
| 3.0 "Nagareyama" | 2020-12 | Standalone `dotnet` tool; emits ES modules directly, Babel dependency dropped[^2]. |
| 4.0 "Snake Island" | 2023 | Multi-target: TypeScript, Python, Rust, Dart (beta) via `--lang`; JSX support. |

## References

[^1]: Fable homepage. https://fable.io/
[^2]: Fable blog (Fable 3 "Nagareyama" announcements, 2020). https://fable.io/blog/
[^3]: SAFE Stack — full-stack F# (Saturn + Azure + Fable + Elmish). https://safe-stack.github.io/
[^4]: Fable README, FCS fork note. https://github.com/ncave/fsharp/pull/2
[^5]: Fable REPL (fable-standalone in the browser). https://fable.io/repl/
[^6]: Fable docs, .NET/F# compatibility. https://fable.io/docs/
[^7]: ts2fable — generate Fable bindings from TypeScript declarations. https://github.com/fable-compiler/ts2fable

## Tags

fsharp, javascript, typescript, python, rust, dart, compiler, transpiler, dotnet, functional-programming, frontend
