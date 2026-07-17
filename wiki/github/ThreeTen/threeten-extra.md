# ThreeTen/threeten-extra

> Date-time classes that `java.time` deliberately left out of the JDK — extra chronologies, typed period amounts, intervals, and TAI/UTC scientific time.

[GitHub repo](https://github.com/ThreeTen/threeten-extra) ·
[Official website](https://www.threeten.org/threeten-extra/) ·
[License: BSD-3-Clause](https://github.com/ThreeTen/threeten-extra/blob/main/LICENSE.txt)

## Overview

ThreeTen-Extra is a companion library to `java.time`, the JSR-310 date-time API that shipped in Java SE 8. It is maintained by Stephen Colebourne, who was the JSR-310 specification lead and the original author of Joda-Time[^1]. The premise is stated plainly in the README: "Not every piece of date/time logic is destined for the JDK. Some concepts are too specialized or too bulky to make it in." This library is where those rejected-but-useful classes live.

Because the same person designed both `java.time` and this library, the APIs are consistent by construction — the same immutability, the same `Temporal`/`TemporalAmount` interfaces, the same fluent `with`/`plus`/`minus` conventions. It is effectively an official extension in everything but JDK membership. The defining tradeoff is exactly that non-membership: you take on a third-party dependency (BSD-3-Clause, zero transitive deps, Java 8+) to get classes that feel like they should have been in the standard library but were cut for scope. For most teams that is a clean deal; the friction is cultural (reviewers ask "why not just `java.time`?") rather than technical.

It is a small, stable, single-maintainer library. 423 stars and 81 forks reflect a utility that is depended on widely but rarely discussed — the kind of dependency that sits transitively under larger frameworks. Releases are infrequent but ongoing (1.9.0 and 1.10.0 both landed in 2026 after a roughly two-year gap following 1.8.0), and the code is heavily tested. Treat it as "done and curated," not "abandoned."

## Getting Started

Maven:

```xml
<dependency>
  <groupId>org.threeten</groupId>
  <artifactId>threeten-extra</artifactId>
  <version>1.10.0</version>
</dependency>
```

Gradle:

```groovy
implementation 'org.threeten:threeten-extra:1.10.0'
```

```java
import org.threeten.extra.Interval;
import org.threeten.extra.YearQuarter;
import org.threeten.extra.AmountFormats;
import java.time.*;
import java.util.Locale;

// A half-open instant interval [start, end)
Interval q = Interval.of(
    Instant.parse("2026-01-01T00:00:00Z"),
    Instant.parse("2026-04-01T00:00:00Z"));
boolean inside = q.contains(Instant.parse("2026-02-15T12:00:00Z")); // true

// Quarter-of-year as a first-class type
YearQuarter yq = YearQuarter.of(2026, 1);          // 2026-Q1
LocalDate quarterEnd = yq.atEndOfQuarter();        // 2026-03-31

// Human-readable period/duration formatting, localized
String words = AmountFormats.wordBased(
    Period.of(0, 2, 3), Duration.ofHours(5), Locale.ENGLISH);
// "2 months, 3 days and 5 hours"
```

## Architecture / How It Works

The library is a flat set of value types built on the same abstractions as `java.time`. There is no runtime, no configuration, no service layer — every class is immutable, `Serializable`, and thread-safe. It groups into five families:

- **Typed period amounts** — `Days`, `Weeks`, `Months`, `Years`, `Hours`, `Minutes`, `Seconds`. Each wraps a single `int`/`long` count and implements `TemporalAmount`, so `date.plus(Days.of(3))` type-checks in a way that a raw `Period` cannot. `PeriodDuration` composes a `Period` and a `Duration` into one amount — the thing `java.time` pointedly refuses to provide because mixing date-based and time-based units is ambiguous across DST.
- **Extra date types** — `YearQuarter`, `Quarter`, `YearWeek` (ISO week-based), `DayOfMonth`, `DayOfYear`, `OffsetDate`. These fill gaps in the `Year`/`YearMonth`/`MonthDay` progression that `java.time` shipped only partway through.
- **`Interval`** — a pair of `Instant`s representing a half-open span on the instant timeline, with `contains`, `overlaps`, `encloses`, `abuts`, `union`, `intersection`, `span`. This is the direct descendant of Joda-Time's `Interval`.
- **Alternative chronologies** — `CopticChronology`, `EthiopicChronology`, `JulianChronology`, `BritishCutoverChronology` (Julian-to-Gregorian 1752 cutover), `DiscordianChronology`, `InternationalFixedChronology`, `PaxChronology`, `Symmetry010Chronology`, `Symmetry454Chronology`, and `AccountingChronology` (configurable 4-4-5 / 4-5-4 fiscal calendars). Each plugs into `java.time.chrono.Chronology`, so a `ChronoLocalDate` in the Coptic calendar interoperates with the standard API.
- **Scientific time** — `TaiInstant`, `UtcInstant`, and `UtcRules` model International Atomic Time (TAI) and UTC with explicit leap-second handling, which `java.time`'s `Instant` deliberately ignores (it assumes 86,400-second days).

`Temporals` and `AmountFormats` are the utility odds and ends: extra `TemporalAdjuster`s (e.g. next/previous working day) and localized word-based rendering of periods and durations across dozens of locales. The whole thing ships as a JPMS module, `org.threeten.extra`.

## Production Notes

**Leap-second data is baked in and goes stale.** The `UtcRules` / `TaiInstant` machinery relies on a compiled-in table of historical leap seconds. New leap seconds (announced by the IERS) require a library upgrade to be recognized. In practice no positive leap second has been added since 2016 and the metrology community has voted to phase them out by 2035, so this is a slow-moving risk — but if you compute TAI-UTC offsets for dates near a future leap second on an old jar, you will be wrong. Pin to a recent release for any TAI/UTC work.

**`Interval` is instant-only.** It operates on the `Instant` timeline, not on `LocalDate` or `LocalDateTime`. There is no first-class "range of local dates" type here — a recurring request in the issue tracker. For date ranges you either wrap `LocalDate` boundaries yourself or reach for a different library. Do not expect a general-purpose interval/range type.

**`PeriodDuration` does not normalize across the date/time boundary.** By design it keeps months, days, and seconds separate because "1 month" has no fixed number of seconds. Arithmetic that mixes the two applies them in a defined order against a specific temporal; there is no context-free "total duration." This is correct behavior, but it surprises people who expect `PeriodDuration` to collapse to a single `Duration`.

**Alternative chronologies are correctness tools, not display tools.** They compute calendar arithmetic correctly, but formatting/parsing and locale data for, say, the Ethiopic or Discordian calendars are thin. If you need full CLDR-quality localization of a non-ISO calendar, verify the output before shipping.

**Single-maintainer cadence.** This is not a knock — it is one of the most carefully maintained libraries in the Java ecosystem — but release timing follows one person's availability. The 1.8.0 (2024) to 1.9.0 (2026) gap is normal for a library that is essentially feature-complete. Don't file "is this dead?" issues; do budget for the fact that a niche feature request may sit for a while. Commercial support and coordinated security disclosure are available via a Tidelift subscription[^2].

**No transitive dependencies.** Adding it cannot drag in a supply-chain surprise; the jar is self-contained and requires only Java 8+.

## When to Use / When Not

**Use when:**
- You need a quarter, ISO week, or half-open instant interval as a real type instead of hand-rolled fields.
- You want type-safe single-unit amounts (`Days.of(n)`) rather than stringly-typed `Period`/`Duration`.
- You need a non-ISO calendar system (Coptic, Ethiopic, Julian/British-cutover, or 4-4-5 accounting) that plugs into `java.time`.
- You need leap-second-aware TAI/UTC instants for scientific or metrology work.
- You want localized "2 hours and 30 minutes" style rendering of durations.

**Avoid when:**
- Plain `java.time` already covers your case — most business apps never need anything here, and the extra dependency is friction reviewers will question.
- You need a general date-range type over `LocalDate` (this library only intervals `Instant`).
- You're on a platform where adding any dependency is costly and you can inline the one helper you need.
- You need heavy localized formatting of exotic calendars — the chronology support is arithmetic-first.

## Alternatives

- JodaOrg/joda-time — the predecessor by the same author; use it only on pre-Java-8 codebases, otherwise `java.time` + this library supersedes it.
- Just `java.time` (JDK built-in) — use instead when the standard `Year`/`YearMonth`/`Duration`/`Period` types already meet your needs and you want zero dependencies.
- google/guava — use its `Range`/`ClosedRange` when you want a generic range abstraction over comparable types rather than a date-specific `Interval`.
- ThreeTen/threetenbp — use this backport instead when you are stuck on Java 6/7 and need `java.time` itself (not the extras).
- unitsofmeasurement/indriya (JSR-385) — use instead when your real need is general typed quantities/units, not calendar-specific amounts.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2012-11-18 | Started alongside JSR-310 as the "extras" staging ground[^1]. |
| 0.8 | 2014-02-10 | Early pre-1.0 line tracking the Java 8 release. |
| 0.9 | 2014-12-09 | Last 0.x release. |
| 1.0 | 2017-02-26 | First stable, SemVer-committed release[^3]. |
| 1.4 | 2018-08-20 | Continued chronology and amount-type expansion. |
| 1.5.0 | 2019-02-24 | — |
| 1.6.0 | 2021-02-18 | — |
| 1.7.0 | 2021-08-01 | — |
| 1.8.0 | 2024-04-16 | — |
| 1.9.0 | 2026-05-31 | Resumed releases after a ~2-year gap. |
| 1.10.0 | 2026-06-16 | Current release; Java 8+, no dependencies[^3]. |

## References

[^1]: Stephen Colebourne — author of Joda-Time and JSR-310 (java.time) specification lead; ThreeTen-Extra project home. https://www.threeten.org/threeten-extra/
[^2]: ThreeTen-Extra README — support and security disclosure via Tidelift. https://github.com/ThreeTen/threeten-extra#support
[^3]: Release history / tags — dates from the GitHub releases API for ThreeTen/threeten-extra. https://github.com/ThreeTen/threeten-extra/releases

## Tags

java, date-time, jsr-310, calendar, chronology, temporal, jdk8, library, leap-seconds, interval, colebourne
