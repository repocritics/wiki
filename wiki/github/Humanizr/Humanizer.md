# Humanizr/Humanizer

> A .NET library of extension methods that turn strings, enums, dates, timespans, numbers, and quantities into human-readable text (and some of it back again).

[GitHub repo](https://github.com/Humanizr/Humanizer) ·
[NuGet](https://www.nuget.org/packages/Humanizer) ·
[License: MIT](https://github.com/Humanizr/Humanizer/blob/main/license.txt)

## Overview

Humanizer is the de facto "display formatting" utility for the .NET ecosystem. It is a collection of extension methods — `"PascalCase".Humanize()`, `DateTime.UtcNow.AddHours(-2).Humanize()`, `"person".Pluralize()`, `3501.ToWords()` — that convert machine-shaped values into text a person would read. The string-humanization core was originally extracted from the BDDfy testing framework, where test method names were turned into readable specifications[^1]. It is a .NET Foundation project and ships under the MIT license[^2] (GitHub reports the license as unrecognized only because the file is named `license.txt`; the text is standard MIT, copyright ".NET Foundation and Contributors").

The library's defining tension is that human-readable is culture-dependent. Almost every method has a locale dimension, and by default Humanizer reads `CultureInfo.CurrentUICulture` at call time. The same code produces "2 hours ago" on one machine and "il y a 2 heures" on another depending on thread culture — convenient for localized UIs, a recurring source of surprise in tests and servers where culture is not what the developer assumed. The second tension is coverage versus size: to support dozens of locales Humanizer historically shipped a large graph of satellite assemblies, which v3 restructured (see History).

## Getting Started

```bash
dotnet add package Humanizer
```

```csharp
using Humanizer;

// Strings
"PascalCaseInput".Humanize();            // "Pascal case input"
"Pascal case input".Dehumanize();        // "PascalCaseInput" (lossy inverse)
"Long text to truncate".Truncate(10);    // "Long text…"

// Enums (DescriptionAttribute wins when present)
UserType.SuperAdmin.Humanize();          // "Super admin"

// Dates & timespans (relative, localized)
DateTime.UtcNow.AddHours(-2).Humanize();          // "2 hours ago"
TimeSpan.FromMilliseconds(1299630020).Humanize(3); // "2 weeks, 1 day, 1 hour"

// Numbers & inflection
3501.ToWords();       // "three thousand five hundred and one"
1.Ordinalize();       // "1st"
"person".Pluralize(); // "people"
1999.ToRoman();       // "MCMXCIX"
```

## Architecture / How It Works

Humanizer is not a framework; it is a bag of `static` extension methods over BCL types plus a global `Configurator`. Behavior splits into a few loosely-related subsystems:

- **String transforms.** `Humanize`, `Dehumanize`, `Transform`, `Titleize`, `Truncate`. `Transform` takes `IStringTransformer` implementations (`To.LowerCase`, `To.TitleCase`, …) so casing rules are extensible rather than a fixed enum. `Truncate` takes pluggable `ITruncator` strategies (fixed length, fixed word count, from left/right).
- **Inflector.** English pluralization/singularization driven by a `Vocabularies.Default` rule set of regexes plus irregular and uncountable word lists. It is heuristic and English-only; unknown irregulars must be registered with `AddIrregular` / `AddUncountable`.
- **Date/time humanization.** `DateTime`, `DateTimeOffset`, and `TimeSpan` reduce to relative phrases. The strategy is swappable via `Configurator.DateTimeHumanizeStrategy` — the default picks the largest unit, `PrecisionDateTimeHumanizeStrategy` rounds by a configurable threshold (default .75).
- **Number formatting.** `ToWords`, `ToOrdinalWords`, `ToRoman`, `ToMetric`, `Ordinalize`, and quantity helpers.
- **Localization.** Each locale must resolve eight canonical surfaces (`list`, `formatter`, `phrases`, `number`, `ordinal`, `clock`, `compass`, `calendar`). As of v3, locale authors check in YAML under `src/Humanizer/Locales` and a Roslyn source generator emits the runtime tables into the single package — the older split into per-locale satellite NuGet packages is gone[^3].

Configuration is process-global mutable state (`Configurator.*`). This makes one-line customization easy and makes concurrent or per-request configuration awkward: there is no scoped/DI-native configuration surface, so changing a strategy or collection formatter changes it for the whole process.

## Production Notes

**Culture is the number-one footgun.** Methods default to the ambient `CurrentUICulture`. Output that looks stable on a developer's `en-US` box can change in CI, on a differently-locale'd server, or after a library sets the thread culture. Pin the culture explicitly (`Humanize(culture: "en-US")`, `TimeSpan.Humanize(culture: ...)`) anywhere the output is asserted on or user-visible in a fixed language. Assertions like `Assert.Equal("2 hours ago", …)` are a classic flaky test when culture is not pinned.

**`DateTime.Humanize` defaults `utcDate: true`.** Passing a local `DateTime` without flipping that parameter silently shifts results. Prefer `DateTimeOffset` where the offset is unambiguous. Date humanization is explicitly lossy — there is no `Dehumanize` for dates.

**Inflection is approximate.** `Pluralize`/`Singularize` cover common English plus a curated irregular list, but they will get domain vocabulary wrong (proper nouns, coined terms, non-English words). When correctness matters, register your terms in `Vocabularies.Default` at startup — but remember that mutates global state.

**Packaging weight (pre-v3).** Older 2.x versions pulled a broad set of locale satellite assemblies through the `Humanizer` metapackage, inflating output for apps that only needed English. Teams often referenced `Humanizer.Core` plus specific locales to trim this. v3's single-package generated-locale model changes that calculus; re-check your dependency graph after upgrading.

**Upgrading 2.14.1 → 3.x is a project, not a bump.** v2.14.1 (Jan 2022) was the last stable 2.x, and v3 did not stabilize until late 2025 — roughly a four-year gap during which most of the ecosystem stayed on 2.14.1[^4]. v3 drops target frameworks (net4.6.1–net4.7.2 are no longer officially supported — they can consume the netstandard2.0 asset but "may not work correctly"), restructures locale packaging, and changes some string behaviors: v3 preserves inputs containing no recognized letters (`"@@".Humanize()` returns `"@@"` instead of `""`). Follow the migration checklist in `docs/migration-v3.md`[^3].

**Hot paths.** These are convenience/display helpers, not zero-allocation primitives. Enum humanization uses reflection over attributes; string and number methods allocate. Fine for rendering; avoid calling them per-item inside tight loops over large collections without caching.

## When to Use / When Not

**Use when:**
- You are rendering machine values to people in a .NET app: enum labels, relative timestamps, pluralized counts ("3 items"), number-to-words.
- You want localized relative-time and quantity strings without hand-writing per-locale tables.
- You want readable BDD/test descriptions or generated documentation from identifiers.

**Avoid when:**
- You need deterministic, locale-independent output and don't want to remember to pin culture everywhere.
- You need correct morphology for languages beyond the shipped locales, or reliable inflection of specialized vocabulary.
- You are formatting in a hot path where allocations and reflection cost matters.
- You need correct date/time arithmetic (time zones, calendars) rather than display text — that is a different problem.

## Alternatives

- axuno/SmartFormat — .NET string templating with pluralization, gender, and conditional formatting inside format strings; use it when the logic belongs in a template rather than in fluent extension calls.
- nodatime/NodaTime — use it when the real need is correct date/time and time-zone arithmetic, not human-friendly display (complementary to, not a replacement for, Humanizer).
- python-humanize/humanize — the closest analog in the Python ecosystem; use it when you need the same relative-time/number-to-words behavior outside .NET.
- date-fns/date-fns — JS/TS relative-time formatting (`formatDistanceToNow`); use it when the humanization needs to happen in the browser or Node.
- System.Globalization (built-in) — use raw `CultureInfo`, `TimeSpan.ToString`, and standard numeric/date format strings when you want simple formatting with no extra dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2012-05-03 | Repository created; string humanization extracted from the BDDfy framework[^1]. |
| 2.2 | 2017-05-04 | Mature 2.x line; broad locale coverage via satellite packages[^4]. |
| 2.14.1 | 2022-01-29 | Last stable 2.x release; the version most of the ecosystem ran for years[^4]. |
| 3.0.0-beta.13 | 2024-03-12 | v3 in extended beta; locale generation and TFM changes in progress[^4]. |
| 3.0.1 | 2025-11-20 | First stable 3.x: source-generated locales, single-package model, dropped legacy TFMs, `"@@"` preservation[^3]. |
| 3.0.10 | 2026-03-07 | Latest stable at time of writing; net10.0/net8.0/net48 + netstandard2.0[^4]. |

## References

[^1]: Humanizer README — origin in the TestStack.BDDfy framework and full feature examples. https://github.com/Humanizr/Humanizer/blob/main/README.md
[^2]: Humanizer on NuGet (`dotnet add package Humanizer`), a .NET Foundation project under MIT. https://www.nuget.org/packages/Humanizer
[^3]: v3 migration guide — breaking changes, locale/source-generator restructure, supported frameworks. https://github.com/Humanizr/Humanizer/blob/main/docs/migration-v3.md
[^4]: Humanizer releases and tags (dates for 2.2, 2.14.1, 3.0.x). https://github.com/Humanizr/Humanizer/releases

## Tags

csharp, dotnet, localization, string-manipulation, date-time, pluralization, formatting, i18n, extension-methods, developer-tools
