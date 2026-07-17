# thomasgalliker/ObjectDumper

> A C# utility that serializes in-memory .NET objects to human-readable strings — or to compilable C# initializer code — for debugging and logging.

[GitHub repo](https://github.com/thomasgalliker/ObjectDumper) ·
[NuGet: ObjectDumper.NET](https://www.nuget.org/packages/ObjectDumper.NET/) ·
[License: MIT](https://github.com/thomasgalliker/ObjectDumper/blob/develop/LICENSE)

## Overview

ObjectDumper is a small single-purpose library that turns an arbitrary .NET object graph into a string, primarily so you can drop it into a log file or the debugger's output without writing per-type formatting code[^desc]. It fills the same niche as LINQPad's `Dump()` or Ruby's `pp`: you have an object, you want to see its shape and values, and JSON is either too noisy or loses type information. The package ships on NuGet as `ObjectDumper.NET` and targets a broad range of runtimes going back to the PCL era (Xamarin, UWP, .NET Framework, modern .NET)[^readme].

Its distinguishing feature is a second output mode. `DumpStyle.Console` produces indented, human-oriented text; `DumpStyle.CSharp` produces syntactically valid C# object-initializer code that can be pasted back into a `.cs` file to reconstruct the object[^readme]. That second mode is the reason people reach for this library over a generic pretty-printer — it is useful for capturing a production object graph and turning it into a deterministic test fixture.

The defining tension of the project is licensing. GitHub reports the repository as MIT and the `LICENSE` file in the tree is a verbatim, unmodified MIT license[^license]. The README, however, declares a *different* dual license: "free for non-commercial use" with "commercial use requires a license" via a $50+ donation[^readme]. These two statements are contradictory and both are shipped in the same repository. Consult both before depending on it commercially (see Production Notes).

## Getting Started

```
PM> Install-Package ObjectDumper.NET
```

```csharp
using System;
using System.Collections.Generic;

var persons = new List<Person>
{
    new Person { Name = "John", Age = 20 },
    new Person { Name = "Thomas", Age = 30 },
};

// Human-readable text (default)
Console.WriteLine(ObjectDumper.Dump(persons));

// Compilable C# initializer code
Console.WriteLine(ObjectDumper.Dump(persons, DumpStyle.CSharp));
```

The first call yields indented `Name: "John"` / `Age: 20` lines under a type header; the second yields a `var listOfPersons = new List<Person> { new Person { Name = "John", Age = 20 }, ... };` block that compiles as-is[^readme].

## Architecture / How It Works

The public surface is deliberately tiny: a static `ObjectDumper.Dump(object)` entry point, a `DumpStyle` enum (`Console` / `CSharp`), and a `DumpOptions` type for tuning. Internally the two styles are separate dumper implementations behind a shared traversal.

The traversal walks the object graph via reflection, reading public properties (and optionally fields) and recursing into nested objects, collections, and dictionaries. Because reflection-based graph walking will loop forever on cyclic references, the dumper tracks already-visited object instances and stops recursion when it re-encounters one, and `DumpOptions` exposes a maximum-depth cap to bound very deep graphs. `DumpOptions` also governs cosmetic and semantic behavior — indentation size and character, line-break style, whether to emit only settable properties, property ordering, culture/number formatting, and how types like `DateTime` are instantiated in the C# output.

The `DumpStyle.CSharp` path is the more constrained of the two: to emit code that actually compiles it must decide how to instantiate each value (constructor vs. initializer), how to render literals for primitives/enums/`DateTime`/`Guid`, and how to name the root variable. This is where its behavior is most opinionated and where edge cases (read-only properties, types without a usable public constructor, anonymous types, records) can produce output that does not round-trip cleanly. The `Console` path has no such constraint and is correspondingly more forgiving.

The assembly is strong-named, signed with `ObjectDumper.snk` committed in the repository, so it can be referenced from other signed assemblies[^readme].

## Production Notes

- **The license is genuinely ambiguous.** The MIT `LICENSE` file grants unrestricted commercial use; the README revokes it for commercial users pending a $50+ donation[^license][^readme]. A verbatim MIT grant is generally hard to walk back once published, but this is a legal question, not a technical one. If you ship this in a commercial product, resolve the contradiction with the author before relying on either reading rather than assuming the permissive one.
- **This is a debugging/logging tool, not a serialization format.** Output is not versioned, not guaranteed stable across releases, and not meant to be parsed back (except by hand, in the CSharp mode). Do not use it as a persistence or wire format — use `System.Text.Json` or JSON for that.
- **Reflection cost is real.** Dumping large or deep graphs invokes reflection over every property. It is fine for occasional diagnostic output; it is not something to call on a hot path or per-request in high-throughput code without measuring.
- **CSharp mode does not round-trip everything.** Types with no public parameterless constructor, computed/read-only properties, private state, or non-trivial invariants may emit initializer code that either fails to compile or reconstructs an object that is not equal to the original. Treat generated fixtures as a starting point to review, not a guarantee.
- **Sensitive data leaks by default.** The dumper prints property values verbatim, including passwords, tokens, and PII, straight into your logs. Use `DumpOptions` exclusion settings (or dump curated view-models) before pointing this at anything that touches secrets.
- **Maintenance is single-author and intermittent.** One primary maintainer, low bus factor; issues can sit. The `develop` branch is the default and the integration target — read it, not a stale `master`, when checking current behavior[^branch].

## When to Use / When Not

**Use when:**
- You want a one-liner to see a full object graph in a log file or console during debugging.
- You need to snapshot a live object graph as C# initializer code to seed a unit test.
- You want type-aware, indented output rather than JSON noise.

**Avoid when:**
- You need a stable, parseable serialization format — use JSON (`System.Text.Json` / Newtonsoft).
- You are dumping in a hot path or on large graphs where reflection overhead matters.
- Your organization needs unambiguous licensing for commercial use and cannot get the MIT-vs-README contradiction resolved.
- You need guaranteed round-tripping of complex types via the CSharp output.

## Alternatives

- MoaidHathot/Dumpify — colorized, table-style console dump aimed purely at readable debugging output; no C#-code emission.
- JamesNK/Newtonsoft.Json — when you actually want a parseable, stable format; `Formatting.Indented` covers readable diagnostics.
- dotnet/runtime (System.Text.Json) — the in-box serializer; use it for persistence/wire formats instead of a debug dumper.
- ServiceStack/ServiceStack.Text — includes `.Dump()` / `.PrintDump()` extensions for quick human-readable inspection.
- Choose ObjectDumper specifically when the C# initializer-code output (fixture generation) is the feature you need; otherwise a JSON serializer is usually the more predictable choice.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-06-09 | Repository created; Console-style object serialization[^created]. |
| 2.x+ | post-1.x | `DumpStyle.CSharp` added — emits compilable C# initializer code[^readme]. |
| current | 2026-04-02 | Most recent commit on the `develop` default branch (as of this writing)[^pushed]. |

## References

[^desc]: GitHub repository description, `thomasgalliker/ObjectDumper`. https://github.com/thomasgalliker/ObjectDumper
[^readme]: Project README (`ObjectDumper.NET`), incl. `DumpStyle.Console` / `DumpStyle.CSharp` examples, strong-name key, and license section. https://github.com/thomasgalliker/ObjectDumper/blob/develop/README.md
[^license]: `LICENSE` file — verbatim MIT License, "Copyright (c) 2026 Thomas Galliker". GitHub reports SPDX `MIT`. https://github.com/thomasgalliker/ObjectDumper/blob/develop/LICENSE
[^created]: GitHub API `created_at` for `thomasgalliker/ObjectDumper`: 2017-06-09.
[^pushed]: GitHub API `pushed_at`: 2026-04-02; default branch `develop`.
[^branch]: GitHub API `default_branch`: `develop`.

## Tags

csharp, dotnet, debugging, logging, serialization, object-dump, reflection, developer-tools, nuget, diagnostics
