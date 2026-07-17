# natemcmaster/CommandLineUtils

> Command-line argument parsing, input validation, and console utilities for .NET — a maintained fork of Microsoft's abandoned `Microsoft.Extensions.CommandLineUtils`.

[GitHub repo](https://github.com/natemcmaster/CommandLineUtils) ·
[Official website](https://natemcmaster.github.io/CommandLineUtils/) ·
[License: Apache-2.0](https://github.com/natemcmaster/CommandLineUtils/blob/main/LICENSE.txt)

## Overview

CommandLineUtils is a library for building console applications in .NET: parsing
arguments and options, defining subcommands, validating user input, and generating
`--help` text. It ships as the NuGet package `McMaster.Extensions.CommandLineUtils`,
which is a distinct namespace from the original `Microsoft.Extensions.CommandLineUtils`
it forked from[^1].

Its history explains its character. The original was written inside the ASP.NET Core
tooling repo, then abandoned by Microsoft[^2]. Nate McMaster forked it in 2017 and
extended it for four years — an attribute-based API, generic typed options, dependency
injection hooks, validators, and console interaction helpers — before declaring the
project in maintenance mode in 2022[^3]. Since then only critical bugs (notably security
issues) are fixed; the API surface is intentionally frozen. The recent commit timestamps
reflect dependency and CI upkeep, not feature work.

The defining tension is stability versus direction. The library is mature, well-documented,
and unlikely to break under you — but it is explicitly not the future of .NET CLI parsing.
Microsoft's own `System.CommandLine` (`dotnet/command-line-api`) is the officially-blessed
successor, and new greenfield projects face a genuine choice between a stable dead-end and
an official-but-longer-churning alternative. For many small tools, "stable dead-end" is the
correct engineering answer.

## Getting Started

```
$ dotnet add package McMaster.Extensions.CommandLineUtils
```

The attribute API binds options to properties on a class:

```c#
using System;
using McMaster.Extensions.CommandLineUtils;

public class Program
{
    public static int Main(string[] args)
        => CommandLineApplication.Execute<Program>(args);

    [Option(Description = "The subject")]
    public string Subject { get; } = "world";

    [Option(ShortName = "n")]
    public int Count { get; } = 1;

    private void OnExecute()
    {
        for (var i = 0; i < Count; i++)
            Console.WriteLine($"Hello {Subject}!");
    }
}
```

The builder API constructs the same app imperatively — useful when options are dynamic:

```c#
var app = new CommandLineApplication();
app.HelpOption();
var subject = app.Option("-s|--subject <SUBJECT>", "The subject", CommandOptionType.SingleValue);
var count = app.Option<int>("-n|--count <N>", "Repeat", CommandOptionType.SingleValue);
app.OnExecute(() =>
{
    for (var i = 0; i < (count.HasValue() ? count.ParsedValue : 1); i++)
        Console.WriteLine($"Hello {subject.Value() ?? "world"}!");
});
return app.Execute(args);
```

## Architecture / How It Works

`CommandLineApplication` is the central object: it holds the option/argument definitions,
the subcommand tree, and the parse/execute pipeline. Both public APIs are thin layers over
it. The attribute API uses reflection at startup to walk `[Option]`, `[Argument]`,
`[Command]`, and `[Subcommand]` attributes on a model type and build the same
`CommandLineApplication` the builder API exposes directly — so the two are not separate
engines, just two front doors to one parser.

Options are strongly typed through `CommandOption<T>` with a value-parser abstraction, so
`-n 5` can bind to an `int` and surface a parse error rather than a silent default.
Validation is layered on top via `System.ComponentModel.DataAnnotations` attributes plus a
fluent validator builder, and validation runs before your execute handler fires.

Subcommands are recursive `CommandLineApplication` instances, which is why help generation,
option inheritance, and unrecognized-argument handling behave consistently at any depth.
The attribute API can wire a subcommand hierarchy from nested model classes.

Beyond parsing, the package bundles console utilities that are independently useful:
`Prompt` (yes/no, password masking, typed input with defaults), `ArgumentEscaper`
(correct quoting when spawning child processes), `DotNetExe` (locating the `dotnet` host),
and the `IConsole` / `IReporter` abstractions that let you swap `System.Console` for a test
double. Dependency injection is supported through a `Microsoft.Extensions.DependencyInjection`
integration so command classes can take constructor-injected services.

The package multi-targets **.NET 8.0** and **.NET Framework 4.7.2** rather than
`.NET Standard`, following Microsoft's post-.NET-5 guidance that two concrete targets are
simpler than a Standard facade[^4].

## Production Notes

- **Maintenance mode is real, not rhetorical.** Feature requests and non-critical bugs will
  not be addressed; the maintainer has been explicit that only security-class fixes land[^3].
  Treat the current API as the final API. This is a feature if you value stability and a
  liability if you expect the library to grow with .NET.
- **Reflection at startup.** The attribute API's convenience costs a reflection scan on each
  run. For most tools this is negligible, but it matters for AOT/trimming and for
  cold-start-sensitive scenarios. There is no source generator; if you need Native AOT
  friendliness, the builder API avoids some (not all) reflection, and `System.CommandLine`
  is the better-aligned choice.
- **Namespace collision trap.** `Microsoft.Extensions.CommandLineUtils` (the abandoned
  original) still exists on NuGet with overlapping type names. Copy-pasted samples or stale
  answers can pull in the wrong package; confirm you are on
  `McMaster.Extensions.CommandLineUtils`[^1].
- **`OnExecute` return conventions.** A handler returning `int` becomes the process exit
  code; `void` implies `0`. Async handlers use `OnExecuteAsync`. Mixing these up silently
  returns `0` on paths you expected to fail — verify exit codes in CI for scripts that gate
  on them.
- **Old-TFM drop-off.** The 4.x line raised the minimum targets; projects still on very old
  .NET Framework or `netcoreapp` versions may be pinned to an earlier major and will not
  receive even the security backports going forward.

## When to Use / When Not

**Use when:**
- You want a stable, well-documented parser for a small-to-medium .NET CLI and value a
  frozen API over an evolving one.
- You like the attribute-based, declarative model and want validation, subcommands, and help
  generation with minimal ceremony.
- You need the bundled console helpers (`Prompt`, `ArgumentEscaper`, `DotNetExe`) as much as
  the parser itself.

**Avoid when:**
- You are starting a long-lived project that should track Microsoft's official CLI direction
  — reach for `System.CommandLine` instead.
- You need Native AOT / aggressive trimming with zero reflection overhead.
- You expect upstream to add features, tab-completion generators, or new .NET-version support
  over time.

## Alternatives

- dotnet/command-line-api — Microsoft's official `System.CommandLine`; use it when you want the actively-developed, AOT-friendly, source-generator path and tab completion.
- commandlineparser/commandline — long-standing `CommandLineParser`; use it for simple attribute-based option binding without a rich subcommand/DI story.
- spectreconsole/spectre.console — `Spectre.Console.Cli` plus rich rendering; use it when styled output, tables, and interactive prompts matter as much as parsing.
- mayuki/Cocona — minimal-API-style CLI framework (ASP.NET-minimal-APIs feel); use it when you want the least boilerplate and convention-driven commands.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017-08 | Forked from `Microsoft.Extensions.CommandLineUtils` after Microsoft abandoned it[^1][^2]. |
| 2.x | 2018 | Attribute-based API, generic typed options, validation, DI integration added. |
| 3.x | ~2020 | Continued feature work; DataAnnotations validation and console utility expansion. |
| 4.x | ~2022 | Target-framework modernization (concrete TFMs over .NET Standard)[^4]. |
| maint. | 2022 | Declared maintenance mode — security-only fixes going forward[^3]. |

## References

[^1]: README — "Project origin and status." https://github.com/natemcmaster/CommandLineUtils#project-origin-and-status
[^2]: aspnet/Common issue — Microsoft's abandonment of the original package. https://github.com/aspnet/Common/issues/257
[^3]: Project status discussion. https://github.com/natemcmaster/CommandLineUtils/issues/485
[^4]: README — "Supported .NET Versions"; Microsoft .NET Standard guidance. https://learn.microsoft.com/en-us/dotnet/standard/net-standard

## Tags

csharp, dotnet, command-line, cli, argument-parsing, console, nuget, dotnet-core, maintenance-mode
