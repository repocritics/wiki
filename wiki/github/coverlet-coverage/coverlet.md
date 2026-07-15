# coverlet-coverage/coverlet

> Cross-platform code coverage for .NET, built on IL instrumentation — the default coverage tool bundled with the `dotnet` test templates.

[GitHub repo](https://github.com/coverlet-coverage/coverlet) ·
[License: MIT](https://github.com/coverlet-coverage/coverlet/blob/master/LICENSE)

## Overview

Coverlet is a code coverage library for .NET that measures line, branch, and method coverage across .NET Framework (Windows) and modern .NET on every supported platform[^1]. It is a [.NET Foundation](https://dotnetfoundation.org/) project, MIT-licensed, and has been the de facto open-source coverage tool for the ecosystem since OpenCover — its Windows-only predecessor — went dormant. As of 2026 it has ~3,200 stars and ~400 forks, is actively maintained (last push July 2026), and is referenced by default in the `dotnet new xunit` test template, so most .NET teams use it without ever choosing it explicitly[^1].

The defining characteristic — and the source of most of its footguns — is that Coverlet works by **rewriting compiled IL**. Before a test run it locates the referenced assemblies that ship PDBs, injects hit-recording instructions at each sequence point, runs the tests, then restores the original assemblies and reads the recorded hits[^1]. This makes it portable and runner-agnostic, but it also means coverage accuracy is bounded by what the C# compiler emits: async/await state machines, iterator methods, and other compiler-generated code produce phantom branches that keep branch-coverage percentages below 100% for correct code.

The other defining tension is packaging. Coverlet ships as **four separate drivers** for four different integration points, they are mutually exclusive per test project, and picking the wrong one is the single most common support issue. The project's own documentation reflects the current `master`, not the released NuGet packages, so the changelog is the authority on whether a documented feature actually exists in the version you installed[^1].

## Getting Started

The most common path is the VSTest data collector — already referenced in xunit test projects:

```bash
dotnet add package coverlet.collector
dotnet test --collect:"XPlat Code Coverage"
# → TestResults/<guid>/coverage.cobertura.xml
```

The MSBuild driver instruments during the build instead, and is convenient for thresholds and inline options:

```bash
dotnet add package coverlet.msbuild
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura \
            /p:Threshold=80 /p:Exclude="[xunit.*]*"
```

Do **not** add both `coverlet.collector` and `coverlet.msbuild` to the same test project. For standalone or integration tests that spawn a separate process, use the global tool (`coverlet.console`) or, on the Microsoft Testing Platform, the newer `coverlet.MTP` extension[^1].

## Architecture / How It Works

Coverlet is an IL rewriter built on Mono.Cecil. Its pipeline is deliberately simple:

1. **Select** — walk the test assembly's references and keep the ones with a matching PDB (no PDB, no source mapping, no instrumentation).
2. **Instrument** — for each sequence point, insert a call that records a hit into a shared memory-mapped/temporary file. Backing-field and compiler-generated constructs are filtered out via a `CecilSymbolHelper` so that auto-properties and the like are not double-counted[^1].
3. **Run** — the unmodified test runner executes the now-instrumented assemblies on disk.
4. **Restore + report** — original assemblies are copied back, hits are read, and results are emitted as `json`, `cobertura`, `lcov`, `opencover`, or `teamcity`.

The four drivers wrap this same engine at different lifecycle points, which is why their capabilities differ:

- **coverlet.collector** — a VSTest `DataCollector`. Instruments in-process during `dotnet test`; the mainstream, best-supported path.
- **coverlet.msbuild** — an MSBuild task chained after the test target. Runs inside the build, which is why it struggles with tests that fork child processes.
- **coverlet.console** — a .NET global tool that instruments a built assembly and then invokes a `--target` runner; the escape hatch for integration tests and non-standard runners.
- **coverlet.MTP** — a native extension for the [Microsoft Testing Platform](https://learn.microsoft.com/dotnet/core/testing/microsoft-testing-platform-intro), the successor architecture to VSTest[^1].

Because instrumentation mutates files on disk and restores them afterward, an interrupted run (killed process, crashed runner) can leave instrumented binaries behind — a known failure mode discussed in the project's Known Issues[^2].

## Production Notes

- **VSTest can stop the process early.** A long-standing issue: `dotnet test` sometimes tears down the host before Coverlet flushes hits, yielding empty or partial reports. The collector and MSBuild drivers both inherit it; the console tool sidesteps it. This is documented but not fully solved[^2].
- **Async/iterator branch coverage is misleading.** Compiler-generated state machines create branches that no test can reach, so realistic branch coverage plateaus well under 100%. Teams typically gate on line coverage and treat branch numbers as directional, not absolute.
- **Deterministic builds need a workaround.** With `ContinuousIntegrationBuild=true` / deterministic source paths, Coverlet cannot map hits back to source without extra `SourceRootMappings` / `CoverletGetPathMap` configuration. The maintainers describe the current solution as non-optimal[^3].
- **MTP vs. VSTest is now a hard incompatibility.** `coverlet.collector` and `coverlet.msbuild` rely on VSTest and do **not** work under the Microsoft Testing Platform. With .NET 10, `dotnet test` enables MTP by default, so those drivers silently produce nothing unless you set `<TestingPlatformDotnetTestSupport>false</TestingPlatformDotnetTestSupport>` or migrate to `coverlet.MTP`[^1]. This is the sharpest upgrade cliff in the project's history.
- **Exclusion is filter-string driven.** Include/exclude use `[AssemblyFilter]TypeFilter` glob syntax (e.g. `[*.Tests]*`), plus the `[ExcludeFromCodeCoverage]` attribute. Getting the filters wrong quietly under- or over-reports rather than erroring.
- **It only measures — it doesn't visualize.** Coverlet emits machine formats; you pair it with ReportGenerator for HTML, or a wrapper like Fine Code Coverage inside Visual Studio.
- **Governance risk.** The original author (tonerdo) and several co-maintainers are marked inactive in the README; day-to-day maintenance rests on a small remaining group[^1]. The project is stable and .NET Foundation-backed, but the bus factor is low.

## When to Use / When Not

**Use when:**
- You want cross-platform coverage for SDK-style .NET projects with zero cost.
- You already run `dotnet test` (the collector is likely referenced already).
- You need standard formats (cobertura/lcov/opencover) to feed Codecov, Coveralls, SonarQube, or ReportGenerator.
- You want thresholds enforced in CI without extra tooling.

**Avoid / reconsider when:**
- Your projects are legacy non-SDK `.csproj` — Coverlet only supports SDK-style projects[^1].
- You target the Microsoft Testing Platform and cannot adopt `coverlet.MTP` yet.
- You depend on precise branch coverage over async-heavy code — expect structural noise.
- You want an IDE-integrated, vendor-supported experience — dotCover or Microsoft's own coverage may fit better.

## Alternatives

- OpenCover/opencover — the older PDB-based instrumentation coverage tool; use only for legacy .NET Framework where Coverlet's SDK-style requirement blocks you (largely unmaintained).
- SteveGilham/altcover — alternative instrumentation-based coverage with heavier configurability and PowerShell/Fake integration; use when you need finer control than Coverlet exposes.
- microsoft/vstest (Microsoft.CodeCoverage / `--collect "Code Coverage"`) — Microsoft's first-party, now cross-platform coverage driven by `.runsettings`; use when you want vendor support over community tooling.
- danielpalme/ReportGenerator — not a collector but the standard companion; use it alongside Coverlet to turn cobertura output into HTML/badges.
- JetBrains dotCover (commercial) — use when you want tight IDE integration and are willing to license it.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Project created | 2018-01 | IL-instrumentation coverage as a cross-platform OpenCover successor[^1]. |
| coverlet.collector | ~2019 | VSTest data collector added to work around MSBuild-driver limits; later referenced by default in `dotnet new` test templates. |
| .NET Foundation project | — | Adopted the .NET Foundation Code of Conduct and governance[^1]. |
| coverlet.MTP | ~2024–2025 | Native extension for the Microsoft Testing Platform, the VSTest successor[^1]. |
| .NET 10 MTP default | 2025 | `dotnet test` defaults to MTP, breaking `collector`/`msbuild` drivers unless opted out[^1]. |

## References

[^1]: Coverlet README, coverlet-coverage/coverlet (master). https://github.com/coverlet-coverage/coverlet
[^2]: Coverlet Known Issues. https://github.com/coverlet-coverage/coverlet/blob/master/Documentation/KnownIssues.md
[^3]: Coverlet Deterministic build support. https://github.com/coverlet-coverage/coverlet/blob/master/Documentation/DeterministicBuild.md

## Tags

dotnet, csharp, code-coverage, testing, instrumentation, cross-platform, msbuild, vstest, il-rewriting, mono-cecil, dotnet-foundation
