# Cysharp/ConsoleAppFramework

> A C# CLI framework that generates the entire argument-parsing entry point at compile time via a Source Generator, leaving effectively zero framework overhead at runtime.

[GitHub repo](https://github.com/Cysharp/ConsoleAppFramework) ·
[License: MIT](https://github.com/Cysharp/ConsoleAppFramework/blob/master/LICENSE)

## Overview

ConsoleAppFramework is a command-line application framework for .NET, maintained by Cysharp — the studio behind MessagePack-CSharp, UniTask, and ZString, led by Yoshifumi Kawai (neuecc)[^1]. It positions itself as "Zero Dependency, Zero Overhead, Zero Reflection, Zero Allocation, AOT Safe," and the current major line (v5) delivers on those adjectives by doing all of its work in a Roslyn Incremental Source Generator rather than at runtime[^2].

The defining idea is unusual for the .NET ecosystem. Most CLI libraries (System.CommandLine, CommandLineParser, Spectre.Console.Cli) parse a schema at startup using reflection or a fluent builder. ConsoleAppFramework instead inspects the lambda or method you hand to `ConsoleApp.Run(args, ...)` at compile time and emits a concrete, specialized `Run` overload whose body already knows every option name, type, and default. The generated code is close to what a careful engineer would hand-write: a `switch` over argument names calling `int.TryParse` and friends inline, with help text baked in as string constants. There is no framework object graph to construct, no dictionary of handlers, and — for the common path — no allocation[^2].

The tradeoff is that this power lives entirely inside the compiler. Behavior is driven by the shapes of your delegates and by XML documentation comments (used for descriptions and aliases), not by a runtime API you can introspect or unit-test in isolation. Debugging means reading generated source, and the framework requires a recent toolchain: .NET 8 and C# 13 (LangVersion 13) at minimum[^2]. It is a strong fit for single-shot CLI tools and Native AOT binaries, and a poor fit if you want a stable, reflection-driven API or need to run on older runtimes.

## Getting Started

```bash
dotnet add package ConsoleAppFramework
```

The package ships only an analyzer — it adds no DLL reference. On first use the generator emits an internal `ConsoleAppFramework.ConsoleApp` class into your assembly.

```csharp
using ConsoleAppFramework;

// invoked as: ./cmd --foo 10 --bar 20  ->  "Sum: 30"
ConsoleApp.Run(args, (int foo, int bar) => Console.WriteLine($"Sum: {foo + bar}"));
```

```csharp
// Multiple + nested commands via the builder
var app = ConsoleApp.Create();
app.Add("echo", (string msg) => Console.WriteLine(msg));
app.Add("sum",  (int x, int y) => Console.WriteLine(x + y));
app.Run(args);
```

Parameter names become `--lower-kebab-case` options (`jsonValue` → `--json-value`), matched case-insensitively. Return types may be `void`, `int`, `Task`, or `Task<int>`; a returned `int` sets `Environment.ExitCode`[^2].

## Architecture / How It Works

Unlike typical source generators that key off attributes, ConsoleAppFramework analyzes the *body* of the delegate passed to `Run`/`RunAsync` and generates a matching `Run` method whose signature and implementation are specialized to that delegate. The README frames this as "function-like macros" in the Rust sense: the call site expands into real code rather than dispatching through a generic runtime[^2].

Two usage modes exist. The direct `ConsoleApp.Run(args, handler)` path is for a single command. For multiple or nested commands you call `ConsoleApp.Create()` to obtain a `ConsoleAppBuilder`, then `Add(name, handler)`, `Add<T>()` (each public method of `T` becomes a command), or `[RegisterCommands]` on a class. Nested commands are declared by space-separated paths (`app.Add("foo bar baz", ...)`), and both commands and options support `|`-separated aliases[^2].

Because the command set is fixed at compile time, the builder does not use a runtime dictionary. The generator emits one strongly-typed field per registered command (`Action`, `Action<int,int>`, `Func<..., Task>`, etc.) and a generated `switch` on the command name that casts the incoming delegate with `Unsafe.As<T>`. Help output is embedded as `const` strings rather than assembled on demand, so `--help` and `--version` cost essentially nothing[^2].

Feature modules are pulled in only when used, and the generator emits the minimal code to support them: `CancellationToken` handling via `PosixSignalRegistration` (SIGINT/SIGTERM for graceful shutdown), a filter/middleware pipeline, `System.ComponentModel.DataAnnotations` validation, constructor-injection DI, `Microsoft.Extensions` (Logging/Configuration) integration, high-performance parsing through `ISpanParsable<T>`, params-array and JSON argument parsing, and `--` escape handling[^2]. For `static` methods you can even pass a managed function pointer (`&Method`) so the generated dispatch uses a `delegate* managed<...>` with no delegate allocation at all[^2].

## Production Notes

- **Toolchain floor is real.** On .NET 8 you must set `<LangVersion>13</LangVersion>` explicitly, or the generator will not compile the code it emits[^2]. This surprises teams pinned to older SDKs.
- **Generator timing in IDEs.** Recent Visual Studio runs source generators on save or at compile time; the README notes that stale or "type not found" errors after editing often clear after one manual compile, or by setting Source Generators to "Automatic" in the C# advanced options[^2]. Expect occasional friction where the generated `ConsoleApp` symbol appears missing until a rebuild.
- **Documentation lives in comments.** Because lambdas and local functions cannot carry XML doc comments in C#, any command that needs descriptions or option aliases must be a method on a class (typically via `Add<T>` or `[RegisterCommands]`)[^2]. Migrating from inline lambdas to documented commands is a structural change, not a cosmetic one.
- **Debuggability.** When parsing behaves unexpectedly, the answer is in the generated file, not a library you can step into with familiar frames. Enabling `EmitCompilerGeneratedFiles` to inspect output is close to mandatory for non-trivial setups.
- **AOT is the intended target.** The zero-reflection design pays off most under Native AOT and for cold-start CLI tools, where reflection/IL-emit caching strategies never warm up. On a long-running host process the runtime savings are far less meaningful.
- **Signals without a token.** If a command does not take a `CancellationToken` parameter, SIGINT/SIGTERM are not intercepted and the process terminates immediately — graceful shutdown is opt-in per command[^2].

## When to Use / When Not

**Use when:**
- Building single-shot CLI tools where startup time and binary size matter, especially with Native AOT.
- You want reflection-free, allocation-light argument parsing that reads like hand-written code.
- You are already on .NET 8+/C# 13 and comfortable with source-generator-driven development.

**Avoid when:**
- You must run on .NET Framework or pre-.NET 8 runtimes, or cannot raise `LangVersion` to 13.
- You want a runtime-introspectable, unit-testable parser API decoupled from the compiler.
- You need rich terminal UI (tables, prompts, progress, colors) — that is a different problem than argument parsing.
- Source-generator opacity (debugging generated code) is a dealbreaker for your team.

## Alternatives

- spectreconsole/spectre.console — use instead when you need rich terminal rendering (tables, prompts, spinners) alongside command parsing.
- dotnet/command-line-api — use System.CommandLine when you want Microsoft's official parser and broader runtime support, accepting a heavier, reflection/builder model.
- commandlineparser/commandline — use when you need a mature, widely-adopted, reflection-based attribute parser that runs on older frameworks.
- natemcmaster/CommandLineUtils — use for a lightweight attribute/builder CLI library with a long track record on legacy runtimes.
- mayuki/Cocona — use when you like the "methods become commands" ergonomics but want a Generic Host-based, reflection-driven design rather than a source generator.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-02-21 | Repository created; early versions built atop `Microsoft.Extensions.Hosting` (Generic Host) with reflection-based command binding[^3]. |
| v5 | 2024 | Full rewrite as a Roslyn Incremental Source Generator: zero dependency, zero reflection, Native AOT safe; requires .NET 8 / C# 13[^2]. |

## References

[^1]: Cysharp organization (Yoshifumi Kawai / neuecc). https://github.com/Cysharp
[^2]: ConsoleAppFramework README (v5), Cysharp. https://github.com/Cysharp/ConsoleAppFramework
[^3]: ConsoleAppFramework releases and tag history. https://github.com/Cysharp/ConsoleAppFramework/releases

## Tags

csharp, dotnet, cli, command-line, source-generator, native-aot, argument-parsing, roslyn, zero-allocation, console-app
