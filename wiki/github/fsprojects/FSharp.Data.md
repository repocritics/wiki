# fsprojects/FSharp.Data

> F# type providers for JSON, XML, CSV, and HTML — compile-time types inferred
> from sample data, plus the parsers and HTTP helpers to back them at runtime.

[GitHub repo](https://github.com/fsprojects/FSharp.Data) ·
[Official website](https://fsprojects.github.io/FSharp.Data) ·
[License: Apache-2.0](https://github.com/fsprojects/FSharp.Data/blob/main/LICENSE.md)

## Overview

FSharp.Data is the canonical demonstration of F#'s type provider mechanism:
point `JsonProvider` (or `XmlProvider`, `CsvProvider`, `HtmlProvider`) at a
sample document, and the compiler synthesizes a strongly-typed API for that
data shape — field names become properties, `"42"` becomes `int`, missing
fields become `option`. It began in 2012 alongside F# 3.0's type provider
feature, with Tomas Petricek and Gustavo Guerra as its early authors; the
design was later written up as a PLDI paper[^1]. Current maintainers are Don
Syme and Phillip Carter[^2].

The defining tension is *types from samples*: you get static typing over
schemaless data with zero hand-written model classes, but the guarantee is
only as good as the sample. If production data diverges from the sample, the
failure surfaces at runtime, not compile time. Schema-based inference (XSD
for XML; JSON Schema since 6.5.0, March 2025[^3]) narrows this gap where
schemas exist.

At 870 stars it looks small next to mainstream .NET libraries, but that
undersells it: this is one of the longest-established F# community packages
(repo created December 2012) and a default dependency in F# scripting and
data-exploration workflows. It is F#-only by construction — the provided
types are invisible to C#.

## Getting Started

```bash
dotnet add package FSharp.Data
```

```fsharp
open FSharp.Data

// Types inferred at compile time from an inline sample
[<Literal>]
let sample = """{ "name": "FSharp.Data", "stars": 870, "topics": ["json"] }"""
type Repo = JsonProvider<sample>

let repo = Repo.Parse """{ "name": "AnotherRepo", "stars": 12, "topics": [] }"""
printfn "%s has %d stars" repo.Name repo.Stars   // typed access, no casts

// CSV with typed rows; HTTP helper included
type Stocks = CsvProvider<"stocks-sample.csv">
let body = Http.RequestString("https://api.github.com/repos/fsprojects/FSharp.Data")
```

Samples can be inline literals, local file paths, or URLs. Prefer local files
committed to the repo — a URL sample is a network dependency of your build.

## Architecture / How It Works

Type providers are compiler plugins: `FSharp.Data.DesignTime.dll` (built on
FSharp.TypeProviders.SDK) is loaded by the F# compiler and by IDE tooling at
design time. It parses the sample, runs structural inference (unifying types
across records, widening `int` → `decimal` → `string`, marking fields absent
in some samples as optional), and emits provided types.

The provided types are **erased**: they do not exist as real .NET types in
the compiled assembly. `repo.Name` compiles down to a lookup on the runtime
representation — `JsonValue` for JSON, `XmlElement` wrappers for XML,
`CsvRow` for CSV, `HtmlNode` for HTML. Consequences: no reflection over
provided types, no serialization frameworks against them, no visibility from
C#, and they make poor public API surface for a distributable library.

The runtime half is an ordinary library that stands on its own: a JSON parser
(`JsonValue`), an RFC 4180 CSV parser, an error-tolerant HTML parser with
full named-entity handling, a `Http` module for requests, and a WorldBank
provider that types a live remote dataset. Since 8.1.0 the `HtmlProvider`
also surfaces schema.org microdata and JSON-LD blocks as typed containers[^3]
— useful for structured scraping of pages that embed them.

Inference is tunable via static parameters: `SampleIsList` (treat sample as a
list of representative examples), `Culture` (number/date parsing),
`PreferOptionals`, `PreferDateOnly` (7.0), `PreferDateTimeOffset` (8.1),
`InferTypesFromValues`.

## Production Notes

**Samples are build inputs.** A URL sample means every compile — including
CI and every IDE background type-check — hits the network. Flaky endpoint,
flaky build. Vendor sample files locally and keep them representative.

**Missing-field behavior is a silent footgun.** By default, accessing a
non-optional field that is absent at runtime returns a default (`""` for
strings, `NaN` for floats) rather than throwing. `ExceptionIfMissing` (added
8.1.0) opts into fail-fast behavior; it defaults to `false` for backward
compatibility[^3]. Audit this if data quality matters.

**Compile-time cost scales with sample size.** Large samples slow every
type-check in the editor, not just full builds. Trim samples to structurally
representative excerpts.

**Design-time tooling is a moving part.** The design-time DLL must load
inside your IDE's process; historically this caused editor-specific bugs
(e.g. a file lock preventing Visual Studio 2022 from closing, fixed in
6.4.0[^3]). After .NET SDK or editor updates, provider red-squiggles that
disappear on rebuild are usually design-time load issues, not code errors.

**XML security posture deserves a look.** An XXE hardening fix shipped in
6.6.0 was reverted in 7.0.1; 8.0.0 instead added an explicit `DtdProcessing`
static parameter[^3]. If you parse untrusted XML, set it deliberately.

**Release cadence changed sharply in 2026.** After a quiet stretch (6.4.1 in
October 2024, 6.5.0 in March 2025), versions 6.6.0, 7.0, and 8.0.0 all
shipped within one week of February 2026, followed by near-weekly 8.1.x
patches[^3]. Much of this stream is authored via "Repo Assist," an AI
assistant the repo uses for triage and proposed fixes[^2] — mostly
allocation/performance work plus the occasional behavioral fix. Maintenance
is active (last push July 2026, 8 open issues), but pin versions and read
release notes before jumping majors; three majors in a year is unusual churn
for a 13-year-old library.

## When to Use / When Not

**Use when:**
- You write F# and consume JSON/CSV/XML/HTML whose shape you know by example
  — scripts, ETL, data exploration, API clients without published schemas.
- You want typed access without writing and maintaining DTO classes.
- You have an XSD or JSON Schema and want compile-checked access code.
- You scrape HTML and benefit from typed tables, microdata, or JSON-LD.

**Avoid when:**
- The consuming code is C# or a mixed-language public API — erased types
  don't cross that boundary; define concrete DTOs instead.
- Data shape is volatile or adversarial: sample-based types give false
  confidence; explicit decoders fail more legibly.
- You need high-throughput serialization of your own domain types — this is
  a reading/inference library, not a serializer.
- Build determinism is paramount and you can't vendor samples locally.

## Alternatives

- thoth-org/Thoth.Json — explicit decoder/encoder combinators for F# (and
  Fable); use instead when you want hand-written, legible failure handling
  over inferred magic.
- Tarmil/FSharp.SystemTextJson — System.Text.Json support for F# records,
  unions, and options; use instead when you define your own types and need
  fast (de)serialization interoperable with the .NET ecosystem.
- JamesNK/Newtonsoft.Json — reflection-based serialization to concrete
  classes; use instead for C#-shared models or legacy .NET codebases.
- AngleSharp/AngleSharp — standards-compliant HTML5 DOM with CSS selectors;
  use instead when scraping needs full spec-level parsing rather than typed
  table extraction.
- fsprojects/SQLProvider — same type-provider idea pointed at databases; use
  instead when the data source is SQL, not documents.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2012-12 | Repo created; F# 3.0 type provider showcase era[^1]. |
| 2.0 | 2014 | Major rewrite (Gustavo Guerra era). |
| 3.0 | 2018 | .NET Standard 2.0 / .NET Core support. |
| 6.3.0 | 2023-04-30 | Post-6.x stabilization line[^3]. |
| 6.4.0 | 2024-03-12 | VS 2022 design-time lock fix[^3]. |
| 6.5.0 | 2025-03-11 | JSON Schema support in JsonProvider[^3]. |
| 6.6.0 | 2026-02-21 | HTML parser perf (~43%), XXE fix (later reverted)[^3]. |
| 7.0 | 2026-02 | `PreferDateOnly` (DateOnly/TimeOnly); 7.0.1 reverts XXE fix[^3]. |
| 8.0.0 | 2026-02-25 | `DtdProcessing`, `PreferFloats`, `With*` copy methods[^3]. |
| 8.1.0 | 2026-03-08 | Microdata + JSON-LD in HtmlProvider, `ExceptionIfMissing`[^3]. |

## References

[^1]: Petricek, Guerra, Syme — "Types from data: Making structured data
    first-class citizens in F#", PLDI 2016.
    https://dl.acm.org/doi/10.1145/2908080.2908115
[^2]: FSharp.Data README (maintainers, Repo Assist, license).
    https://github.com/fsprojects/FSharp.Data#readme
[^3]: FSharp.Data release notes.
    https://github.com/fsprojects/FSharp.Data/blob/main/RELEASE_NOTES.md

## Tags

fsharp, dotnet, type-providers, json, csv, xml, html, data-access, web-scraping, http-client, code-generation
