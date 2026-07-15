# malcommac/SwiftDate

> A Swift toolkit that wraps `Date`, `Calendar`, `TimeZone`, and `DateFormatter` behind an ergonomic value-type API for parsing, math, comparison, and localized display.

[GitHub repo](https://github.com/malcommac/SwiftDate) ·
[License: MIT](https://github.com/malcommac/SwiftDate/blob/master/LICENSE)

## Overview

SwiftDate is a date/time convenience library for the Apple platforms (and Linux via
Foundation), first published in 2015[^1]. Its core proposition is that Foundation's
native date handling — `Date` as an opaque UTC instant, plus separate `Calendar`,
`TimeZone`, `Locale`, and `DateFormatter` objects that must be assembled by hand — is
verbose and error-prone for everyday work. SwiftDate collapses that into two ideas: a
`Region` (calendar + timezone + locale bundled together) and a `DateInRegion` (an instant
interpreted through a Region), plus a large surface of operators, computed properties,
and string helpers layered on top.

The library's reach is broad by design: automatic parsing of ~15 common datetime formats
(ISO8601, RSS, .NET, SQL, HTTP), natural-language math (`2.hours + 5.minutes`), 30+ boolean
comparison helpers (`isToday`, `isNextWeek`, granularity-aware `compare`), derived-date
generation (`dateAt(.startOfMonth)`, `nextWeekday(.friday)`), and relative formatting in
100+ locales. It reports 90% test coverage as of the 5.x line[^2] and, per its own README,
over 3 million CocoaPods downloads[^1].

The defining tension is scope versus modern Foundation. SwiftDate was most valuable when
Foundation's own APIs were painful; since then Apple has shipped `RelativeDateTimeFormatter`
(iOS 13), `Date.FormatStyle` and `Date.ISO8601FormatStyle` (iOS 15), and `Duration` (iOS
16), which cover a meaningful slice of what SwiftDate provides — natively, and without a
dependency. SwiftDate remains broader and more convenient, but the "you need this" case is
narrower than it was in the Swift 3/4 era, and the project's own commit activity has slowed
markedly (last push 2023[^3]).

## Getting Started

Swift Package Manager (add to `Package.swift` dependencies):

```swift
.package(url: "https://github.com/malcommac/SwiftDate.git", from: "7.0.0")
```

CocoaPods: `pod 'SwiftDate'` — or Carthage: `github "malcommac/SwiftDate"`.

```swift
import SwiftDate

// Parse — format auto-detected, or supply your own
let d = "2010-05-20 15:30:00".toDate()!.date

// A Region bundles calendar + timezone + locale
let rome = Region(calendar: .gregorian, zone: .europeRome, locale: .italian)
let event = DateInRegion("2017-01-01 00:00:00", region: rome)!

// Math with time units, comparison, derived dates
let later   = event + 3.months - 2.days
let isSoon  = later.compare(.isThisMonth)
let monthEnd = later.dateAt(.endOfMonth)

// Convert across timezones; format for display
let ny = event.convertTo(region: Region(calendar: .gregorian, zone: .americaNewYork, locale: .english))
print(ny.toFormat("dd MMM yyyy 'at' HH:mm"))          // "31 Dec 2016 at 18:00"
print((Date() - 3.minutes).toRelative())               // "3 minutes ago"
```

## Architecture / How It Works

The type model has two layers. `DateInRegion` is the rich value type: it holds an absolute
`Date` plus a `Region`, and nearly every operation is expressed on it. `Date` itself is also
extended (via extensions like `toFormat`, `+`, `compare`, `dateAt`) so you can use the sugar
directly on Foundation dates — but those operate through an implicit default region
(`SwiftDate.defaultRegion`, itself derived from the device's current calendar/timezone/locale
unless overridden). This dual surface is convenient but is the source of most surprises: an
operation on a bare `Date` and the "same" operation on a `DateInRegion` can disagree if the
default region differs from the one you thought you were in.

Time-unit math (`2.hours`, `3.months`) is built on integer extensions that produce
`DateComponents`, so `date + 3.months` delegates to `Calendar.date(byAdding:)` — meaning it
inherits Foundation's calendar-correct behavior (DST transitions, variable month lengths)
rather than doing naive second arithmetic. Comparison helpers wrap `Calendar.compare(_:_:
toGranularity:)`. Formatting wraps `DateFormatter` with a cache to avoid repeatedly
constructing formatters (a well-known Foundation performance pitfall).

Relative formatting is a self-contained subsystem: the `RelativeFormatter` ships bundled
locale rule tables (derived from the same data lineage as Unicode CLDR / the JS
`javascript-time-ago` project) rather than delegating to Foundation, which is why it supports
100+ locales and multiple styles (`.twitter`, `.default`) independently of OS version. Time
Periods (interval overlap/containment logic) are provided by integrating Matthew York's
DateTools[^4].

## Production Notes

- **Maintenance has stalled.** The last release and the last substantive commits are from
  2023[^3]; open issues sit near 90. It is not archived and still compiles against current
  Swift, but treat it as feature-frozen — do not expect fixes for new-OS edge cases or Swift
  language-mode churn. This is the single most important operational fact.
- **Default-region traps.** Operations on bare `Date` use `SwiftDate.defaultRegion`. If you
  never set it, that's the device locale/timezone — which means the same code produces
  different string output on different users' devices. For anything user-facing or persisted,
  always work through an explicit `DateInRegion` / `Region`, and set `SwiftDate.defaultRegion`
  once at startup if you rely on the `Date` sugar.
- **Formatter cost.** Date formatting is expensive in Foundation; SwiftDate caches formatters,
  but `toRelative` and heavy `toFormat` loops over large arrays still show up in profiles.
  Batch-format or reuse regions rather than constructing a new `Region` per element.
- **`toDate()` auto-detection is a convenience, not a contract.** It tries a fixed list of
  formats in order and returns the first that parses. Ambiguous input (e.g. day/month order)
  can silently parse to the wrong date. In production, pass an explicit format string.
- **Operator overloading is heavy.** `+`, `-`, `>`, `<`, `==` are overloaded across `Date`,
  `DateInRegion`, `DateComponents`, and time units. This aids readability but can slow the
  Swift type-checker in large expressions and occasionally produces confusing overload-
  resolution errors. Break complex one-liners into steps.
- **Migration cost is real.** The 4.x → 5.x transition was a full API rewrite (the
  Region/DateInRegion model was introduced there); code written against 4.x does not port
  mechanically. Pin your major version.

## When to Use / When Not

**Use when:**
- You support older OS targets (below iOS 15 / macOS 12) where `Date.FormatStyle`,
  `RelativeDateTimeFormatter`, and `Duration` are unavailable or thin.
- You do heavy calendar math and comparison and want the readability of `date + 1.month` and
  `date.compare(.isNextWeek)` over hand-assembled `DateComponents`.
- You need relative "3 minutes ago" formatting across many locales with consistent output
  regardless of OS version.

**Avoid when:**
- You target modern OS versions only and your needs are covered by Foundation's own
  `FormatStyle` / `RelativeDateTimeFormatter` / `Duration` — you can drop the dependency.
- You want a dependency under active maintenance with responsive issue turnaround.
- Your app is performance-sensitive around date formatting at scale and you'd rather control
  formatter lifecycle explicitly than through a library's caching.

## Alternatives

- apple/swift-foundation — modern `Date.FormatStyle`, `.ISO8601FormatStyle`,
  `RelativeDateTimeFormatter`, and `Duration` cover much of SwiftDate's surface natively; use
  it when your deployment target is recent and you want zero dependencies.
- davedelong/time — a type-safe calendar library that encodes precision in the type system to
  prevent whole classes of timezone/granularity bugs; use it when correctness-by-construction
  matters more than convenience breadth.
- MatthewYork/DateTools — the Time Periods engine SwiftDate itself embeds; use it directly if
  interval overlap/containment logic is all you need.
- dpreece/DateHelper — a lighter category-style set of `Date` extensions; use it when you want
  parsing/formatting sugar without the Region model or the dependency weight.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2015 | Initial release; `NSDate` category-style helpers[^1]. |
| 4.x | ~2016–2017 | Widely adopted Swift 3 era; classic API before the rewrite. |
| 5.0 | ~2018 | Full rewrite introducing the `Region` / `DateInRegion` model; ~90% test coverage[^2]. |
| 6.x | ~2019 | Swift 5 support, `Codable` conformance for dates and regions. |
| 7.0 | ~2022 | SPM-forward packaging; current major line[^3]. |

(Version dates other than the 2015 creation are approximate; consult the repository's
[releases](https://github.com/malcommac/SwiftDate/releases) and CHANGELOG for exact tags.)

## References

[^1]: SwiftDate README — feature summary and CocoaPods download figure. https://github.com/malcommac/SwiftDate#readme
[^2]: SwiftDate README — "As 5.x the project has 90% of code coverage." https://github.com/malcommac/SwiftDate#readme
[^3]: GitHub API repository metadata (stars, forks, license, `pushed_at` 2023-09-19), retrieved 2026-07-15. https://api.github.com/repos/malcommac/SwiftDate
[^4]: MatthewYork/DateTools — Time Periods integration referenced in the SwiftDate README. https://github.com/MatthewYork/DateTools

## Tags

swift, date-time, timezone, calendar, formatting, parsing, ios, macos, foundation, localization, apple
