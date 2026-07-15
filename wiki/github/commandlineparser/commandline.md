# commandlineparser/commandline

> Attribute-driven command line argument parser for .NET, following the *nix getopt convention.

[GitHub repo](https://github.com/commandlineparser/commandline) ·
[NuGet](https://www.nuget.org/packages/CommandLineParser/) ·
[License: MIT](https://github.com/commandlineparser/commandline/blob/master/License.md)

## Overview

CommandLineParser is a library that turns a decorated C# (or VB.NET, or F#) class into a parsed representation of `string[] args`. You annotate properties with `[Option]`, `[Value]`, and `[Verb]` attributes, hand `args` to `Parser.Default.ParseArguments<T>`, and receive a strongly-typed options object plus an automatically generated `--help` / `--version` screen. It targets the getopt/POSIX style (`-v`, `--verbose`, `--`, clustered short flags) rather than the Windows `/flag` style[^1].

The project began life as `gsscoder/commandline` around 2005 and was rewritten for the 2.x line, which introduced a functional, monadic internal design and broke API compatibility with 1.9.x[^2]. It later moved under the `commandlineparser` organization and is community-maintained. For most of the 2010s it was the default answer to "how do I parse args in .NET," predating Microsoft's own `System.CommandLine`.

The defining tension today is maintenance velocity. The library is stable, widely deployed (tens of millions of NuGet downloads), and does its job — but the repository has seen no pushed commits since early 2024 and carries a large backlog of open issues. New .NET concerns like Native AOT and IL trimming, which its reflection-heavy core does not fully accommodate, are the main reason newer projects look elsewhere.

## Getting Started

```
dotnet add package CommandLineParser
```

```csharp
using CommandLine;

class Options
{
    [Option('v', "verbose", HelpText = "Set output to verbose messages.")]
    public bool Verbose { get; set; }

    [Option('r', "read", Required = true, HelpText = "Input files to be processed.")]
    public IEnumerable<string> InputFiles { get; set; }

    [Value(0, MetaName = "offset", HelpText = "File offset.")]
    public long? Offset { get; set; }
}

class Program
{
    static int Main(string[] args) =>
        Parser.Default.ParseArguments<Options>(args)
            .MapResult(
                opts => { /* run */ return 0; },
                errs => 1);   // errors + --help/--version already printed
}
```

F# support ships as a separate package, `CommandLineParser.FSharp`, which adds handling for `option<'a>` and F# collection types[^3].

## Architecture / How It Works

Parsing is reflection-based. At `ParseArguments<T>` time the library inspects `T`'s attributes to build a specification (which options exist, their arity, defaults, required-ness), tokenizes `args`, and materializes an instance of `T`. It supports both mutable types (public settable properties) and immutable types (constructor-injected), the latter matched by parameter name.

Internally the 2.x rewrite is written in a railway-oriented / monadic style borrowed from the author's `CSharpx` and `RailwaySharp` helpers: parsing produces a sequence of `Error` values rather than throwing, and the public result is a discriminated-union-like `ParserResult<T>` with `Parsed<T>` and `NotParsed<T>` cases. You consume it with `WithParsed` / `WithNotParsed` (side-effecting) or `MapResult` (functional, returns an exit code).

Type mapping is broad: scalars, enums, `Nullable<T>`, `IEnumerable<T>` sequences, and any type with a single-string constructor (so `System.Uri` and similar work without custom converters). Verbs — `git commit`-style subcommands — are modeled as separate attribute-decorated classes passed together to `ParseArguments<Add, Commit, Clone>`, then dispatched with a multi-arm `MapResult`. A default verb is supported.

Help generation (`HelpText.AutoBuild`) reads the same attribute metadata to render usage, wraps text to console width, and can be customized or localized. This is the feature that historically distinguished the library from hand-rolled parsing: the help screen and the parser never drift apart because both derive from one source of truth.

## Production Notes

**Native AOT / trimming.** The reflection- and attribute-driven core is not trim-safe or AOT-friendly out of the box. Under `PublishTrimmed` or `PublishAot` you can get trimming warnings and runtime failures where option types get stripped. This is the single most common reason to prefer `System.CommandLine` or `Cocona` for new self-contained/AOT deployments. There is no source-generator path in the shipped releases.

**Maintenance status.** Treat this as a mature, feature-complete library in low-activity maintenance rather than an actively developed one. The default branch has not received pushed commits since 2024-02, and the issue tracker backlog is in the hundreds. Bugs you hit are unlikely to be fixed upstream quickly; budget for pinning a version and, if necessary, vendoring or patching.

**Immutable-type gotchas.** Constructor-based (immutable) options match by parameter name against option long-names; a rename mismatch fails silently or at runtime rather than at compile time. Records work but require care around the generated constructor.

**Help/version exit behavior.** `Parser.Default` writes help and version output to stderr and treats `--help`/`--version` as a `NotParsed` result. Teams frequently trip on this by returning a non-zero exit code for a successful `--help`, or by wanting help on stdout — both require configuring a custom `Parser` with your own `HelpWriter`.

**Globalization.** Numeric and enum parsing follows the current culture unless configured, which can surprise apps run under non-invariant locales (decimal separators, etc.).

## When to Use / When Not

**Use when:**
- You want declarative, attribute-based options with a free, always-consistent `--help` screen.
- You are on .NET Framework or a JIT-compiled .NET app where AOT/trimming is not a constraint.
- You need verbs, sequences, enums, and immutable option types with minimal ceremony.
- You value a stable, long-lived API and are comfortable pinning a version.

**Avoid when:**
- You ship Native AOT or aggressively trimmed self-contained binaries.
- You want a library with active upstream maintenance and fast bug turnaround.
- You prefer a fluent/imperative builder or a minimal-API surface over attributes.
- You need Microsoft-aligned tooling (tab completion, `dotnet` integration), where `System.CommandLine` fits better.

## Alternatives

- dotnet/command-line-api (System.CommandLine) — Microsoft's own parser; better AOT/trimming story and shell completion. Use when targeting modern self-contained/AOT apps.
- spectreconsole/spectre.console — CLI parsing plus rich console rendering (tables, prompts, progress). Use when the tool's UX matters as much as parsing.
- natemcmaster/CommandLineUtils — attribute-and-builder hybrid, actively maintained fork lineage. Use when you want a similar attribute model with more upkeep.
- mayuki/Cocona — minimal-API style built on the .NET generic host and DI. Use when you want method-as-command with built-in dependency injection.
- adamabdelhamed/PowerArgs — attribute-based with interactive features. Use when you want tab completion and REPL-style interaction on Windows-centric tools.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.9.x | ~2005–2014 | Original `gsscoder/commandline` line; different API surface[^2]. |
| 2.0 | 2017-03 | Functional/monadic rewrite; `ParserResult`, verbs, immutable types[^2]. |
| 2.8.0 | 2020-11 | Broadened target frameworks; Source Link, `snupkg` symbols[^1]. |
| 2.9.1 | 2022-02 | Last widely-used release on the 2.9 line[^4]. |
| — | 2024-02 | Last pushed commit to `master` as of this writing[^5]. |

## References

[^1]: Command Line Parser Library README. https://github.com/commandlineparser/commandline/blob/master/README.md
[^2]: v1.9.x compatibility note and rewrite lineage; original repo `gsscoder/commandline`, tag `stable-1.9.71.2`. https://github.com/gsscoder/commandline/tree/stable-1.9.71.2
[^3]: `CommandLineParser.FSharp` package. https://www.nuget.org/packages/CommandLineParser.FSharp/
[^4]: NuGet package version history. https://www.nuget.org/packages/CommandLineParser/
[^5]: Repository metadata via GitHub API (`pushed_at` 2024-02-29), retrieved 2026-07. https://github.com/commandlineparser/commandline

## Tags

csharp, dotnet, command-line, cli, argument-parser, getopt, fsharp, vb-net, nuget, attributes
