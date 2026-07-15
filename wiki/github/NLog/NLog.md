# NLog/NLog

> Configuration-driven logging for .NET: route log events to files, consoles, databases, and the network through XML or code, without recompiling.

[GitHub repo](https://github.com/NLog/NLog) ·
[Official website](https://nlog-project.org) ·
[License: BSD-3-Clause](https://github.com/NLog/NLog/blob/dev/LICENSE.txt)

## Overview

NLog is one of the two long-standing logging frameworks for .NET, alongside log4net, and predates Microsoft's own logging abstraction by roughly a decade. The project began in 2006[^1] and migrated to GitHub in 2012[^2]. Its defining idea is that logging *routing* — which events go where, in what format — is configuration data, not code: a `NLog.config` XML file (or `appsettings.json` section) declares targets, rules, and layouts that can be changed and hot-reloaded without touching the application binary.

The audience is server-side and desktop .NET developers who want fine control over log output: multiple destinations, per-logger level filtering, structured and templated messages, and a large catalog of output "targets" (file, console, database, mail, network, cloud sinks) supplied as separate NuGet packages. With ~6.5k stars and ~1.4k forks it is smaller in GitHub reach than Serilog but has a deep install base across enterprise .NET and is actively maintained — commits land regularly and 6.0 shipped in 2025[^3].

The central tension is age versus modernity. NLog carries a mature, XML-first configuration model designed before `Microsoft.Extensions.Logging` (MEL) and structured logging existed, then retrofitted both. It works well with the modern `ILogger<T>` abstraction, but you inherit two overlapping mental models — NLog's own logger/target/rule pipeline and MEL's provider/category/scope model — and the seams between them are where most confusion lives.

## Getting Started

```bash
dotnet add package NLog
# ASP.NET Core / generic host integration:
dotnet add package NLog.Web.AspNetCore   # or: NLog.Extensions.Logging
```

```csharp
using NLog;

var logger = LogManager.GetCurrentClassLogger();

// Structured / templated message — {OrderId} and {Customer} are named holes.
logger.Info("Order {OrderId} placed by {Customer}", 42, "alice");

try { /* ... */ }
catch (Exception ex) { logger.Error(ex, "Checkout failed"); }
```

```xml
<!-- NLog.config — copied to output, hot-reloaded if autoReload="true" -->
<nlog xmlns="http://www.nlog-project.org/schemas/NLog.xsd"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      autoReload="true" throwConfigExceptions="true">
  <targets>
    <target name="file" xsi:type="File" fileName="logs/${shortdate}.log"
            layout="${longdate} ${level:uppercase=true} ${logger} ${message} ${exception:format=tostring}" />
  </targets>
  <rules>
    <logger name="*" minlevel="Info" writeTo="file" />
  </rules>
</nlog>
```

## Architecture / How It Works

Four concepts do all the work:

- **Loggers** — obtained via `LogManager.GetCurrentClassLogger()` (name = the calling class's full type name) or `GetLogger("name")`. Loggers are cheap, cached by name, and carry no configuration themselves.
- **Targets** — the destinations. `FileTarget`, `ConsoleTarget`, `NetworkTarget`, and hundreds of community/official targets in separate packages. Wrapper targets (`AsyncWrapper`, `BufferingWrapper`, `RetryingWrapper`, `FallbackGroup`) decorate other targets to add batching, async dispatch, and failover.
- **Rules** — `<logger>` entries match a logger *name pattern* to a minimum level and a `writeTo` target list. Rules are evaluated top-to-bottom; `final="true"` stops further matching, which is the mechanism for muting noisy namespaces.
- **Layouts and Layout Renderers** — a layout is a template string; renderers like `${longdate}`, `${level}`, `${message}`, `${exception}`, `${mdlc:item=...}` (context data) expand at write time. Layouts are usable anywhere a value is needed, including inside `fileName`, so a single File target can fan out to per-day or per-level files.

An event flows: `logger.Info(...)` → level check against matching rules (fast; the common short-circuit is a level filter, not string formatting) → build `LogEventInfo` → pass through each writeTo target and its wrappers → render layout → emit. Structured logging (message templates with `{Named}` holes, and `{@obj}` for destructuring) has been first-class since NLog 4.5[^4]; the captured properties are available to JSON layouts and to targets that persist structured data.

Configuration can also be built entirely in code (`new LogFactory()` / `LoggingConfiguration`), which is the recommended path for AOT and for tests. On modern hosts, `NLog.Extensions.Logging` registers NLog as an `ILoggerProvider` so `ILogger<T>` calls are bridged into the NLog pipeline; category names become NLog logger names.

## Production Notes

- **Async is opt-in and can drop messages.** Wrapping a target in `AsyncWrapper` (or `async="true"`) decouples logging from the request thread, but the default `overflowAction` is `Discard`: under sustained load the queue fills and events are silently dropped. Set `overflowAction="Block"` (backpressure) or `Grow` deliberately, and size `queueLimit`/`batchSize` for your throughput.
- **Silent misconfiguration is the classic footgun.** By default NLog swallows its own errors so a broken config never crashes your app — and never logs anything either. In development set `throwConfigExceptions="true"`, and enable **internal logging** (`internalLogFile` / `internalLogLevel`) to see why a target is not writing. This is the first thing to check when "logs just aren't appearing."
- **`autoReload` watches the config file**, not `appsettings.json`; with the JSON-based setup you reload differently. File-watching also has caveats in containers and on some network shares.
- **File target concurrency.** `keepFileOpen="true"` is faster but assumes a single writer; `concurrentWrites` for multi-process access to one file is slow and best avoided — give each process its own file via a `${processid}`/`${machinename}` layout instead. Archival (`archiveEvery`, `maxArchiveFiles`, `archiveAboveSize`) is powerful but its interaction with open handles is a common source of surprises.
- **The MEL bridge changes semantics.** Going through `Microsoft.Extensions.Logging` means MEL log levels, category names, and `BeginScope` map onto NLog concepts; some NLog-native features (e.g. certain context renderers, `Trace`/`Off` nuances) behave differently than when calling NLog directly. Pick one primary API and be consistent.
- **AOT / trimming.** Full NativeAOT support arrived in NLog 6.0 (2025)[^3]; earlier versions relied on reflection for config parsing and target instantiation, which trimming can break. For trimmed/AOT apps, prefer programmatic configuration and verify with the trim analyzers.

## When to Use / When Not

**Use when:**
- You want output routing controlled by editable config that ops can change without a rebuild.
- You need many destinations, per-namespace level control, failover/buffering, and rich file archival out of the box.
- You are on classic ASP.NET/.NET Framework or a mixed estate where NLog's long history and broad target catalog matter.

**Avoid when:**
- You prefer an all-code, sink-composition model with a strong structured-first ethos — Serilog fits that better.
- Your app only needs simple console/file logging under `ILogger<T>` — the built-in `Microsoft.Extensions.Logging` providers may be enough with no third-party dependency.
- You want a minimal-dependency AOT app on an older NLog major — reflection-based config is a liability before 6.0.

## Alternatives

- serilog/serilog — code-first, structured-logging-native "sinks" model; the most common modern .NET choice. Prefer it when you want fluent configuration and destructuring-first design.
- apache/logging-log4net — the other veteran .NET framework; similar XML-config philosophy. Choose it mainly for parity with existing log4net estates.
- dotnet/runtime (Microsoft.Extensions.Logging) — the abstraction plus built-in providers. Use alone when your needs are simple and you want zero external deps.
- open-telemetry/opentelemetry-dotnet — when logs are one signal in a traces+metrics story and you want vendor-neutral export rather than a logging framework per se.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | ~2006 | Original release; XML-configured targets/rules[^1]. |
| 2.0 | 2011 | Rewrite; async targets, wider platform support. |
| 4.0 | 2015 | Consolidated packaging, config improvements. |
| 4.5 | 2018 | First-class structured logging / message templates[^4]. |
| 5.0 | 2022 | Modernized packaging around netstandard2.0, cleanup of legacy APIs[^5]. |
| 6.0 | 2025 | NativeAOT support; further trimming/AOT-friendly changes[^3]. |

## References

[^1]: NLog project history — nlog-project.org. https://nlog-project.org/
[^2]: GitHub repository metadata (NLog/NLog), created 2012-09-12; ~6.5k stars, ~1.4k forks, BSD-3-Clause, default branch `dev`, last pushed 2026-07. https://github.com/NLog/NLog
[^3]: "List of major changes in NLog 6.0" — 2025-04-29. https://nlog-project.org/2025/04/29/nlog-6-0-major-changes.html
[^4]: NLog wiki, "How to use structured logging." https://github.com/NLog/NLog/wiki/How-to-use-structured-logging
[^5]: NLog project news / release archive. https://nlog-project.org/archives/

## Tags

csharp, dotnet, logging, logging-library, structured-logging, xml-config, aot, cross-platform, observability, netstandard
