# Tyrrrz/CliFx

> Class-first .NET framework for command-line apps: declare commands as attributed classes and let source generators wire up parsing, help, and `Main()`.

[GitHub repo](https://github.com/Tyrrrz/CliFx) ·
[NuGet](https://nuget.org/packages/CliFx) ·
[License: MIT](https://github.com/Tyrrrz/CliFx/blob/prime/License.txt)

## Overview

CliFx is a command-line application framework for .NET by Oleksii Holub (Tyrrrz), first published to NuGet around 2020[^1]. It deliberately positions itself as a *complete application framework* rather than an argument parser: a command is a class annotated with `[Command]` and implementing `ICommand`, its inputs are properties annotated with `[CommandParameter]` / `[CommandOption]`, and the framework owns argument parsing, routing, type conversion, help-text generation, and process exit codes. The stated goal is to remove low-level plumbing so the author only writes the command's logic.

The defining tradeoff is opinionation. CliFx gives you one blessed way to structure a CLI — classes and attributes — and very little surface to deviate from it. That yields minimal boilerplate and a consistent help screen for free, but it also means the model is harder to bend when your needs sit outside the attribute vocabulary (custom parsing grammars, non-class command shapes, or deep integration with a host builder). Compared with Microsoft's own `System.CommandLine`, CliFx trades configurability for a smaller, more prescriptive API.

More recent versions lean on C# source generators: commands must be declared `partial` so the generator can extend them with metadata and emit the entry point, replacing the earlier reflection-based discovery. This is what makes the framework compatible with Native AOT and trimming — a meaningful shift for a CLI library, since fast-starting, self-contained single-file tools are a primary use case[^2].

## Getting Started

```bash
dotnet add package CliFx
```

```csharp
using CliFx;
using CliFx.Attributes;
using CliFx.Infrastructure;

// Command types must be `partial` so the source generator can extend them.
[Command(Description = "Calculates the logarithm of a value.")]
public partial class LogCommand : ICommand
{
    [CommandParameter(0, Description = "Value whose logarithm is to be found.")]
    public required double Value { get; set; }

    [CommandOption("base", 'b', Description = "Logarithm base.")]
    public double Base { get; set; } = 10;

    public ValueTask ExecuteAsync(IConsole console)
    {
        console.Output.WriteLine(Math.Log(Value, Base));
        return default; // completed ValueTask — no async work needed
    }
}
```

The source generator discovers commands and produces `Main()` automatically, so no explicit bootstrap is required for the common case. `-h|--help` and `--version` are bound conventionally. Provide your own `Main()` via `CliApplicationBuilder` only when you need to customize configuration.

## Architecture / How It Works

A command's shape is declared entirely through attributes. `[CommandParameter(order)]` binds *positional* arguments; parameter order values must be unique, and only the last parameter may be non-required or sequence-based (an `IReadOnlyList<T>`), because positional binding cannot otherwise decide how many tokens to consume. `[CommandOption(name, short)]` binds *named* arguments, which have no such restriction — any option can be required, optional, or multi-valued, and can declare an `EnvironmentVariable` fallback that fills the value when the flag is omitted.

Type conversion is handled by an `IInputConverter<T>` pipeline. By default the framework infers a converter from the property type: `string` passthrough, `bool.Parse`, enum parsing (numeric or name), any type exposing a static `Parse(string, IFormatProvider?)` (covering `int`, `double`, `DateTime`, `Guid`, and user types), any type with a `.ctor(string)` (e.g. `FileInfo`), `Nullable<T>`, arrays and array-constructible collections. Unsupported types are handled by subclassing `ScalarInputConverter<T>` or `SequenceInputConverter<T>` and pointing the binding's `Converter` at it. Custom converters must have a public parameterless constructor.

The `IConsole` abstraction is the second load-bearing design decision. Commands never touch `System.Console` directly; they receive an `IConsole` that exposes output, error, input streams, colors, and cancellation. In production this is `SystemConsole`; in tests it is `FakeConsole`/`FakeInMemoryConsole`, which capture output and let you assert on it without spawning a process. This is the mechanism that makes CliFx apps unit-testable, and it is the framework's clearest differentiator against parser-only libraries.

Commands can be composed into hierarchies by name (`[Command("log")]`, `[Command("sum")]`), and a nameless command becomes the default/root. The framework routes on the leading argument tokens and renders a grouped help screen listing subcommands. Cancellation flows through interrupt signals into a `CancellationToken` obtained from the console, so long-running commands can shut down gracefully on Ctrl+C.

## Production Notes

- **`partial` is mandatory, and it propagates.** A command type must be `partial`, and if it is nested, *every* enclosing type must be `partial` too. Forgetting this is the most common first-run compile error after migrating from a reflection-era version.
- **Source-generator upgrades can be a hard break.** The move from reflection-based discovery to source generators changed how commands are declared. Applications written against older CliFx do not upgrade transparently — expect to add `partial`, adjust namespaces (attributes and infrastructure types have moved between releases), and rebuild. Pin the major version and read the changelog before bumping.
- **Native AOT is a first-class target but constrains you.** AOT/trimming compatibility is only preserved if *your* converters and command logic are also trim-safe; reflection over your own domain types inside `ExecuteAsync` can still break a published AOT binary. Test the actual `PublishAot` output, not just the JIT build.
- **No dependency injection is built in.** CliFx does not ship a DI container; to inject services you supply a type activator (e.g. delegating to `Microsoft.Extensions.DependencyInjection`) via the application builder. Out of the box, command types are instantiated with a parameterless constructor.
- **Not a shell-completion or TUI toolkit.** CliFx generates help text and parses arguments; it does not produce shell completion scripts or rich interactive prompts. If you need spinners, tables, and prompts, pair it with a rendering library or choose Spectre.Console.Cli instead.
- **Single-maintainer project.** Development is community-funded and driven primarily by one author. It is actively maintained (recent commits on the `prime` default branch), but roadmap and review throughput depend on one person — weigh that for long-horizon dependencies.

## When to Use / When Not

**Use when:**
- You want a small, opinionated framework where a command is just a class and the help screen is free.
- You're shipping a Native AOT / trimmed single-file .NET CLI and need parsing that survives it.
- You value testable commands (the `IConsole` seam) over maximal configurability.
- Your input model fits parameters + options + standard type conversion.

**Avoid when:**
- You need the Microsoft-blessed, framework-integrated option — `System.CommandLine` ships with the .NET tooling story.
- You want rich terminal UI (tables, prompts, progress) as part of the same library — reach for Spectre.Console.
- You need bespoke parsing grammars or a builder-first API rather than attributes.
- A single-maintainer dependency is a governance risk for your project.

## Alternatives

- dotnet/command-line-api — `System.CommandLine`, Microsoft's own builder-based parser; more configurable, more verbose, use when you want the platform-aligned choice.
- spectreconsole/spectre.console — `Spectre.Console.Cli` has a similar class-first command model plus rich rendering; use when you also want tables, prompts, and colored output.
- natemcmaster/CommandLineUtils — attribute or builder API, long-established; use when you want a mature, flexible parser with DI hooks.
- commandlineparser/commandline — declarative POCO + attributes, parser-only; use when you just need argument binding without an app framework.
- mayuki/Cocona — minimal, method-as-command style inspired by ASP.NET minimal APIs; use when you prefer functions over command classes.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-06 | Repository created[^1]. |
| 1.x | ~2020 | First stable line: reflection-based command discovery, `IConsole` abstraction, hierarchical commands. |
| 2.x | ~2021 | Major release with breaking API changes (binding/converter model, directives). |
| recent | 2024–2026 | Source-generator-based discovery, `partial` command types, Native AOT and trimming support, no external dependencies[^2]. |

## References

[^1]: CliFx repository and package metadata (created 2019-06-02; distributed via NuGet as `CliFx`). https://github.com/Tyrrrz/CliFx
[^2]: CliFx README — feature list (Native AOT and trimming compatibility, source-generated `Main()`, `partial` command requirement, no external dependencies), `prime` branch. https://github.com/Tyrrrz/CliFx/blob/prime/Readme.md

## Tags

csharp, dotnet, cli, command-line, framework, argument-parsing, source-generators, native-aot, terminal, developer-tools
