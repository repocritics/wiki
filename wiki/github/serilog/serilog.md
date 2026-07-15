# serilog/serilog

> Structured logging for .NET built on message templates — you log the values, not a pre-formatted string.

[GitHub repo](https://github.com/serilog/serilog) ·
[Official website](https://serilog.net) ·
[License: Apache-2.0](https://github.com/serilog/serilog/blob/dev/LICENSE)

## Overview

Serilog is a diagnostic logging library for .NET, first released in 2013[^1]. Its defining idea is the *message template*: instead of interpolating values into a string before logging, you write `log.Information("Processed {@Position} in {Elapsed} ms", position, elapsedMs)` and Serilog captures `Position` and `Elapsed` as named properties on the event. The rendered text and the structured data are produced from the same template, so a sink can emit either human-readable lines or machine-queryable JSON without you writing the message twice.

Serilog predates and heavily influenced .NET's own structured logging abstractions. Microsoft's `Microsoft.Extensions.Logging` (MEL) borrowed message-template semantics, and Serilog remains the most widely used concrete provider behind that abstraction[^2]. For most teams the practical shape is: code against `ILogger<T>` from MEL, wire Serilog underneath via `Serilog.Extensions.Hosting` / `Serilog.AspNetCore`, and get structured events flowing to sinks.

The core repository is deliberately small. Serilog is a pipeline — capture, enrich, filter, format, emit — and almost everything useful (file output, Seq, Elasticsearch, OpenTelemetry, JSON config) lives in separate `Serilog.Sinks.*`, `Serilog.Enrichers.*`, and `Serilog.Settings.*` packages under the same GitHub org. The defining tension is this ecosystem sprawl: the core is stable and well-designed, but a working setup means assembling and version-aligning several independently released packages.

## Getting Started

```bash
dotnet add package Serilog
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
```

```csharp
using Serilog;

Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .WriteTo.File("log.txt",
        rollingInterval: RollingInterval.Day,
        rollOnFileSizeLimit: true)
    .CreateLogger();

try
{
    var position = new { Latitude = 25, Longitude = 134 };
    Log.Information("Processed {@Position} in {Elapsed} ms", position, 34);
    throw new InvalidOperationException("Oops...");
}
catch (Exception ex)
{
    Log.Error(ex, "Unhandled exception");
}
finally
{
    await Log.CloseAndFlushAsync(); // flush buffered/async sinks before exit
}
```

The `@` operator (`{@Position}`) tells Serilog to *serialize* the object into structured properties rather than call `ToString()`. A bare `{Position}` would capture the string representation instead.

## Architecture / How It Works

A `LoggerConfiguration` builds an immutable `Logger`. Each event flows through a fixed pipeline:

1. **Capture** — the message template is parsed and each argument is bound to a named property, becoming a `LogEvent` (timestamp, level, template, properties, optional exception). Templates are cached after first parse.
2. **Enrich** — `ILogEventEnricher`s attach ambient context: thread id, machine name, and crucially `LogContext` — an `AsyncLocal`-backed stack of properties pushed via `LogContext.PushProperty(...)`, which flows across `await` boundaries so correlation ids survive async call chains.
3. **Filter** — level checks and predicate filters. A disabled level short-circuits before argument capture, which is why logging is cheap when switched off.
4. **Format & emit** — sinks receive the `LogEvent`. Text sinks render it through an `ITextFormatter`; structured sinks (JSON, Seq, OTel) serialize the properties directly.

`Logger` instances are zero-shared-state and thread-safe; the static `Log` class is an optional global convenience wrapper over one. Sub-loggers created via `ForContext<T>()` are cheap and inherit the pipeline.

Sinks are the extension seam. `WriteTo.Sink(...)` accepts any `ILogEventSink`, and the fluent `WriteTo.Console()` / `WriteTo.File()` methods are extension methods shipped by the individual sink packages — importing a sink package is what makes its configuration method appear. `WriteTo.Async(...)` (a separate package) wraps a sink in a background-thread buffer.

## Production Notes

- **Always `CloseAndFlush`.** Async, file, and network sinks buffer. If the process exits without `Log.CloseAndFlush()` / `CloseAndFlushAsync()`, buffered events are lost. This is the single most common Serilog footgun. `Serilog.AspNetCore`'s host integration handles it for you; console apps must do it explicitly.
- **`{@}` serialization is a memory and PII hazard.** Destructuring a large or cyclic object graph with `@` can capture far more than intended and inflate event size. Use `Destructure.ByTransforming<T>()` or `.Destructure.ToMaximumDepth()` to bound it, and never destructure objects carrying secrets — they land in your logs verbatim.
- **Template vs. properties mismatch.** Positional/named parameters must line up with the template. Passing an interpolated `$"..."` string instead of a template silently defeats structured logging and can also inject `{`/`}` that break parsing. Serilog.Analyzers / the built-in analyzers catch some of this at build time.
- **Two-stage init for ASP.NET Core.** The common pattern is a bootstrap `CreateBootstrapLogger()` before the host is built, then `UseSerilog((ctx, cfg) => ...)` reading final config from `appsettings.json`. Skipping the bootstrap logger means startup exceptions log nowhere.
- **Sink and package version drift.** Because sinks, enrichers, and settings packages release independently, an upgrade of `Serilog` core can leave a sink on an incompatible range. Pin versions and expect to bump several packages together across a major.
- **Minimum-level overrides matter for noise.** `.MinimumLevel.Override("Microsoft.AspNetCore", LogEventLevel.Warning)` is near-mandatory in web apps; the framework emits high-volume `Information` events otherwise.

## When to Use / When Not

**Use when:**
- You want structured events (JSON, Seq, Elasticsearch, OpenTelemetry) rather than flat text lines.
- You're on .NET and want the de-facto standard provider behind `Microsoft.Extensions.Logging`.
- You need rich async-aware context propagation (`LogContext`) and per-sink filtering/formatting.

**Avoid when:**
- You only need a handful of debug lines and don't want to assemble and version-align multiple NuGet packages — MEL's built-in console logger may suffice.
- You need traces/metrics as first-class signals, not just logs — reach for OpenTelemetry's SDK directly (Serilog can feed it, but it is a logging library, not a full telemetry pipeline).
- Ultra-low-allocation hot paths where even disabled-level checks and template caching are scrutinized — benchmark against alternatives for your workload.

## Alternatives

- nlog/NLog — mature, config-file-first .NET logger; choose it when XML/declarative configuration and a huge target library matter more than message-template ergonomics.
- dotnet/runtime (Microsoft.Extensions.Logging) — use the framework-native logging abstraction directly when you don't need a specific provider's features and want zero extra dependencies.
- apache/logging-log4net — use when migrating a legacy log4j-style .NET codebase that already depends on log4net conventions.
- open-telemetry/opentelemetry-dotnet — use when logs are one signal among traces and metrics and you want a vendor-neutral telemetry pipeline; often paired with, not instead of, Serilog.
- getsentry/sentry-dotnet — use when the primary need is error aggregation and alerting rather than general structured logging.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2013 | Initial release; message-template-based structured logging[^1]. |
| 2.0 | 2016 | Pipeline redesign, .NET Core / netstandard support, sink packages split out[^3]. |
| 2.5 | 2018 | `Log.CloseAndFlush`, broad ASP.NET Core integration maturity. |
| 3.0 | 2022 | netstandard2.0+ baseline, nullable annotations, API cleanups[^4]. |
| 4.0 | 2024 | `LogEvent`/pipeline updates, `CloseAndFlushAsync`, modern TFM targeting[^5]. |

## References

[^1]: Nicholas Blumhardt, "Serilog" announcement — 2013. https://nblumhardt.com/2013/05/serilog/
[^2]: Serilog integration with Microsoft.Extensions.Logging. https://github.com/serilog/serilog-extensions-logging
[^3]: Serilog 2.0 release notes. https://github.com/serilog/serilog/releases
[^4]: Serilog 3.0 release notes. https://github.com/serilog/serilog/releases/tag/v3.0.0
[^5]: Serilog releases (4.x). https://github.com/serilog/serilog/releases

## Tags

dotnet, csharp, logging, structured-logging, observability, message-templates, diagnostics, nuget, aspnet-core, library
