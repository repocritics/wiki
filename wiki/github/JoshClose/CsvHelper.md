# JoshClose/CsvHelper

> The default .NET library for reading and writing CSV — object mapping over a hand-tuned field parser, with a large surface area and a history of breaking changes.

[GitHub repo](https://github.com/JoshClose/CsvHelper) ·
[Official website](http://joshclose.github.io/CsvHelper/) ·
[License: MS-PL OR Apache-2.0](https://github.com/JoshClose/CsvHelper/blob/master/LICENSE.txt)

## Overview

CsvHelper is a C# library for reading and writing CSV (delimited text) files, maintained by Josh Close since 2009[^1]. Its distinguishing feature is not raw parsing but **object mapping**: `GetRecords<T>()` streams a CSV into a sequence of typed objects, matching columns to properties by header name, and `WriteRecords()` does the reverse. For most .NET developers it is the reflexive default — the NuGet package with the most downloads in the CSV category — which is as much a function of longevity and Google ranking as of technical merit.

The library sits at a specific point on the tradeoff curve. It is far more convenient than hand-rolling `string.Split(',')` (which is wrong for any CSV with quoted fields), and far more flexible than a fixed schema reader: you get type conversion, culture-aware parsing, fluent `ClassMap` configuration, attribute mapping, custom converters, and hooks for bad/missing data. In exchange you pay in throughput — reflection-based mapping is materially slower than the current generation of SIMD/`DbDataReader` parsers — and in API churn. CsvHelper has shipped many major versions with source-breaking changes, and upgrading across a major boundary is rarely a no-op.

The single most encountered fact about CsvHelper is that, since v13 (2020), `CsvConfiguration` **requires an explicit `CultureInfo`** in its constructor[^2]. This was a deliberate response to years of bug reports from developers whose CSVs silently broke when run on a machine with a different decimal separator or list separator (e.g. German locales where `;` is the list separator and `,` is the decimal point).

## Getting Started

```
dotnet add package CsvHelper
```

Reading a CSV into typed objects:

```csharp
using System.Globalization;
using CsvHelper;

public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
    public DateTime Joined { get; set; }
}

using var reader = new StreamReader("people.csv");
using var csv = new CsvReader(reader, CultureInfo.InvariantCulture);

// GetRecords<T> is lazy — it yields as it reads.
var people = csv.GetRecords<Person>().ToList();
```

Writing:

```csharp
using var writer = new StreamWriter("out.csv");
using var csv = new CsvWriter(writer, CultureInfo.InvariantCulture);
csv.WriteRecords(people);
```

By default, columns bind to public properties by matching the CSV header to the property name (case-insensitive). Explicit control comes from `ClassMap`:

```csharp
public sealed class PersonMap : ClassMap<Person>
{
    public PersonMap()
    {
        Map(m => m.Id).Name("person_id");
        Map(m => m.Joined).TypeConverterOption.Format("yyyy-MM-dd");
    }
}
// csv.Context.RegisterClassMap<PersonMap>();
```

## Architecture / How It Works

CsvHelper layers three concerns that can be used independently:

1. **`CsvParser`** — the field/record tokenizer. Reads a `TextReader` character by character, honoring quotes, escapes, embedded newlines, and the configured delimiter. Exposes raw `string[]` records. This is the fast, allocation-conscious core.
2. **`CsvReader` / `CsvWriter`** — the record layer. Wraps the parser and adds header handling, field access by index/name, and the streaming `GetRecords<T>()` / `WriteRecords()` entry points.
3. **Mapping + type conversion** — reflection-driven binding of fields to object members. Auto-mapping inspects the type; `ClassMap` overrides it declaratively; attributes (`[Name]`, `[Index]`, `[Optional]`, `[Format]`, etc.) are a third path. `ITypeConverter` implementations translate `string ↔ T`, with `TypeConverterOptions` (culture, formats, boolean/null tokens) per member.

Configuration flows through `CsvConfiguration` (a record type in current versions), covering delimiter, quoting mode, `HasHeaderRecord`, trimming, comment handling, and the `BadDataFound` / `MissingFieldFound` / `HeaderValidated` delegate hooks that let you decide whether malformed input throws or is swallowed.

The mapping layer is where the coupling and the cost live. Auto-mapping builds and caches an expression-compiled record accessor per type, which amortizes reflection across a run but means the first record of a new type is comparatively expensive. Because binding is convention-driven, a renamed property or a header typo produces a runtime `HeaderValidationException` rather than a compile error — the library trades static guarantees for flexibility.

## Production Notes

**`GetRecords<T>()` is lazy, and that is a footgun.** It `yield`s records as it pulls from the underlying stream. If you dispose the `CsvReader`/`StreamReader` before enumerating, you get an `ObjectDisposedException`; if you return the raw `IEnumerable<T>` out of a method whose `using` block has closed, you get the same. The fix is to materialize inside the scope (`.ToList()`) — at the cost of holding the whole file in memory. For genuinely large files you must consume lazily within the `using` block.

**Throughput is mid-pack, not leading.** CsvHelper's reflection-and-conversion pipeline is convenient but is measurably slower and allocates more than the current fast-parser generation (Sep, Sylvan). If you are parsing hundreds of megabytes or are on a hot path, benchmark against those before assuming CsvHelper is adequate. For the common case (config files, imports, exports of thousands to low-millions of rows) it is fine.

**Culture is not optional and matters.** `CultureInfo.InvariantCulture` is the safe default for machine-readable data. Using a locale-specific culture silently changes decimal separators, date formats, and — critically — the default field delimiter (`CultureInfo.TextInfo.ListSeparator`), which is `;` in much of Europe. Round-tripping data written under one culture and read under another is a classic production incident.

**CSV injection.** When writing values that a spreadsheet program will later open, cells beginning with `=`, `+`, `-`, or `@` can be interpreted as formulas. CsvHelper exposes `InjectionOptions` in the configuration to sanitize these on write; it is **not** on by default, so exports of user-supplied data need it enabled explicitly.

**Major-version upgrades are real work.** The API has been reshaped repeatedly: configuration moved from mutable properties to an immutable record; the required-`CultureInfo` change (v13) broke essentially every call site; class-map and type-converter signatures have shifted. Pin the version, read the migration/changelog before bumping a major, and expect to touch code. The upside is that the project is still actively maintained (last commit mid-2025) with a large contributor base and corporate sponsorship (the .NET on AWS OSS Fund, Microsoft)[^1].

**Error handling is delegate-based.** Bad data does not always throw. `BadDataFound`, `MissingFieldFound`, and `ReadingExceptionOccurred` are configurable callbacks; leaving them at defaults means some malformed input is tolerated silently. For strict ingestion pipelines, wire these to throw.

## When to Use / When Not

**Use when:**
- You want CSV ↔ POCO mapping with minimal ceremony and don't want to own a parser.
- Your CSVs are irregular (custom delimiters, quoting, headers, per-column formats) and you need declarative mapping and custom type converters.
- You need broad .NET version support and a well-trodden, heavily-documented library where Stack Overflow already has your answer.

**Avoid when:**
- Throughput on very large files is the priority — reach for a faster parser and give up some convenience.
- You want compile-time schema guarantees — convention binding fails at runtime.
- Your need is trivial and fixed (a couple of known columns, no quoting) — a small hand-written reader avoids a dependency, though "no quoting" is an assumption CSV loves to break.

## Alternatives

- nietras/Sep — the current throughput leader; SIMD-accelerated, low-allocation, modern API. Use instead when parse speed on large files is the constraint.
- MarkPflug/Sylvan — `Sylvan.Data.Csv` exposes a fast `DbDataReader`; excellent for ADO.NET-style consumption and bulk loading. Use instead when you want `IDataReader` semantics and speed.
- nreco/NReco.Csv — minimal, fast low-level reader/writer with no object mapping. Use instead when you only need field access and want a tiny dependency.
- dotnet/runtime (Microsoft.VisualBasic `TextFieldParser`) — ships in the BOM, no NuGet dependency. Use instead for occasional parsing where a dependency isn't worth it; the API is dated.
- Cesil — source-generator-oriented mapping aiming to cut reflection cost. Use instead when you want compile-time-generated binding, accepting a smaller ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2009-12-30 | Repository created; library begins as a small reader/writer[^1]. |
| 3.0 | ~2017 | Large rewrite of the mapping and configuration model. |
| 13.0 | 2020 | `CultureInfo` becomes a required constructor argument — the most impactful breaking change[^2]. |
| 15–30+ | 2021–2025 | Configuration moved to an immutable record; injection protection, ongoing type-converter and class-map refinements across successive majors. |
| latest | 2025-06-27 | Actively maintained; last push as of this writing[^1]. |

(Exact release dates for individual majors are omitted where not verified; the milestones above are the ones with clear, load-bearing consequences.)

## References

[^1]: JoshClose/CsvHelper repository, README and metadata — 5.2k stars, dual MS-PL / Apache-2.0, sponsored by the .NET on AWS OSS Fund and Microsoft. https://github.com/JoshClose/CsvHelper
[^2]: CsvHelper documentation, "Getting Started" / configuration — `CsvConfiguration` requires a `CultureInfo`. http://joshclose.github.io/CsvHelper/

## Tags

csharp, dotnet, csv, parser, serialization, data-import, file-io, object-mapping, nuget, library
