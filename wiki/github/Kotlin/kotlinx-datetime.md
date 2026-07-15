# Kotlin/kotlinx-datetime

> A multiplatform Kotlin library for date and time, deliberately kept small and built around a hard split between physical instants and civil (calendar) time.

[GitHub repo](https://github.com/Kotlin/kotlinx-datetime) ·
[API reference](https://kotlinlang.org/api/kotlinx-datetime/) ·
[License: Apache-2.0](https://github.com/Kotlin/kotlinx-datetime/blob/master/LICENSE)

## Overview

`kotlinx-datetime` is JetBrains' official multiplatform date/time library for Kotlin, first published in 2019[^1]. It exists because the Kotlin standard library historically shipped no calendar types: on the JVM you could reach for `java.time`, but Kotlin/Native, Kotlin/JS, and Wasm targets had nothing portable. This library provides one API — `LocalDate`, `LocalDateTime`, `LocalTime`, `TimeZone`, `DateTimePeriod`, and friends — that compiles to every Kotlin target.

The defining design choice is a strict boundary between *physical time* (an `Instant`, a point on a monotonic timeline) and *civil time* (year/month/day/hour components that only mean something relative to a time zone)[^2]. The library refuses to provide operations that quietly blur the two. There is no `LocalDateTime.plus(days)`, for instance, because daylight-saving transitions make such arithmetic ill-defined — you are forced to convert to an `Instant`, do arithmetic there with an explicit `TimeZone`, then convert back. This is opinionated and occasionally verbose, but it eliminates a large class of DST and time-zone bugs by construction.

The library is intentionally minimal. It is ISO 8601-only, and internationalization — locale-specific month names, day names, culturally formatted output — is explicitly out of scope[^2]. As of 2026 it is still labelled **Alpha** for API stability[^3], despite wide production use and a near-daily commit cadence; the Alpha label reflects that the public API has changed across minor releases (see History) more than that the code is unproven.

## Getting Started

Add the dependency (published to Maven Central as `org.jetbrains.kotlinx:kotlinx-datetime`):

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-datetime:0.8.0")
}
```

```kotlin
import kotlinx.datetime.*
import kotlin.time.Clock
import kotlin.time.Instant

// Physical "now" (from the stdlib) -> civil components need a TimeZone
val now: Instant = Clock.System.now()
val local: LocalDateTime = now.toLocalDateTime(TimeZone.of("Europe/Berlin"))
println(local.year to local.month)

// Calendar arithmetic is done on Instant, with an explicit zone
val tz = TimeZone.currentSystemDefault()
val inThreeMonths = now.plus(DateTimePeriod(months = 3), tz)

// Dates without time
val birthday = LocalDate(1990, 2, 15)
val daysAlive = birthday.daysUntil(Clock.System.todayIn(tz))
```

## Architecture / How It Works

The library is a thin, opinionated wrapper over each platform's native date/time facilities rather than a from-scratch calendar engine[^4]:

- **JVM** delegates to `java.time` (JSR-310). Types like `LocalDate` map onto their `java.time` counterparts, so behavior and the IANA time-zone database come from the JDK.
- **Kotlin/Native, JS, Wasm** are backed by a port of the [ThreeTen backport](https://www.threeten.org/threetenbp/). On JS and Wasm/JS, time-zone rules specifically come from the [`js-joda`](https://js-joda.github.io/js-joda/) library, which bundles the tz database.

Because the JVM path is `java.time` and the others are a backport of the same design, semantics are consistent across platforms, but the *source of truth for time-zone data differs by target* — the JVM uses the host JDK's tz data, while Native/JS carry their own. This matters when a zone's rules have been updated in one and not the other.

The type system encodes the physical/civil split. `DateTimeUnit` is split into `DateBased` and `TimeBased` subtypes, and `DatePeriod` is a `DateTimePeriod` with zero time components. `LocalDate.plus` only accepts `DateTimeUnit.DateBased`, so adding hours to a bare date is a compile error, not a runtime surprise. Parsing and formatting outside ISO 8601 go through a builder DSL (`LocalDate.Format { ... }`) or, for partial/compound/out-of-bounds data, the `DateTimeComponents` bag of fields — which can hold nominally invalid values like second `60` for leap-second inputs before you normalize them.

A significant recent architectural change: through version 0.6.x the library owned `kotlinx.datetime.Instant` and `kotlinx.datetime.Clock`. In 0.7.0 these were **moved into the Kotlin standard library** as `kotlin.time.Instant` and `kotlin.time.Clock`, since an instant is useful well beyond calendar code[^5]. `kotlinx-datetime` now builds its civil types on top of the stdlib's `Instant`.

## Production Notes

- **The 0.7.0 `Instant` migration is the big footgun.** Code and, worse, *transitive dependencies* written against `kotlinx.datetime.Instant` can fail at runtime with `ClassNotFoundException` after an upgrade[^5]. If a library you depend on still references the old class, you may need the compatibility artifact `0.7.1-0.6.x-compat`, which re-exposes the removed classes. Expect resolution-ambiguity errors when both `import kotlin.time.*` and `import kotlinx.datetime.*` are present — you must import `Instant`/`Clock` explicitly.
- **kotlinx.serialization coupling.** If you serialize `Instant`, you need kotlinx.serialization 1.9.0+ to align with the stdlib `Instant`[^5]. Version-skew here produces confusing serializer errors.
- **Android below API 26 requires core library desugaring**[^3] — enable it in Gradle and use AGP 4.0+. Without it, `java.time`-backed types are unavailable on old devices and you get runtime `NoClassDefFoundError`.
- **Alpha means minor versions break.** The API has changed shape across 0.x releases (e.g. `dayOfMonth` → `day`, `monthNumber` reshuffles, `Instant` relocation). Pin the version and read release notes before bumping; do not treat this like a 1.0 stability guarantee.
- **Tight Kotlin-version coupling.** Recent releases track a specific Kotlin/stdlib version (0.8.0 expects Kotlin `2.3.21`)[^6]. Because `Instant`/`Clock` now live in the stdlib, an old Kotlin plus a new `kotlinx-datetime` can be incompatible.
- **No localization.** Formatting is ISO 8601 and pattern-DSL only. If you need localized, human-facing dates ("mardi 3 février"), you must format them yourself or fall back to platform APIs.

## When to Use / When Not

**Use when:**
- You target multiple Kotlin platforms (KMP: Android + iOS + JS/Wasm) and need one date/time API across all of them.
- You want the physical-vs-civil discipline enforced by the type system rather than by convention.
- Your formatting needs are ISO 8601 or a fixed machine format.

**Avoid when:**
- You are JVM-only and comfortable with `java.time` — the standard library already gives you a richer, 1.0-stable API with no extra dependency.
- You need locale-aware formatting, non-ISO calendars, or fuzzy/natural-language date handling.
- You require a strong API-stability guarantee; the Alpha label is honest about breaking minor releases.

## Alternatives

- java.time (JSR-310) — the built-in choice for JVM-only projects; richer and stable, but not multiplatform. Use it when you never leave the JVM.
- JakeWharton/ThreeTenABP — `java.time` backport for legacy Android; use it when supporting old Android without KMP and without desugaring.
- js-joda/js-joda — date/time for JavaScript/TypeScript; use it when the project is JS-native rather than Kotlin.
- korlibs/klock — an older Kotlin multiplatform date/time library; use it if you want a different API or need targets/features kotlinx-datetime lacks.
- ThreeTen/threetenbp — the backport this library builds on; use it directly only in non-Kotlin JVM contexts needing a pre-Java-8 port.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2020-05 | First public release; core `LocalDate`/`LocalDateTime`/`Instant`/`TimeZone` types[^1]. |
| 0.4.0 | 2022 | Format/parse improvements, API refinements. |
| 0.6.0 | 2024 | Format-builder DSL and `DateTimeComponents` maturation. |
| 0.7.0 | 2025 | `Instant`/`Clock` moved to `kotlin.time` in the stdlib; compat artifact introduced[^5]. |
| 0.8.0 | 2026 | Current release line; tracks Kotlin `2.3.21`[^6]. |

*Exact patch dates vary; consult the GitHub releases page for authoritative timestamps.*

## References

[^1]: Kotlin/kotlinx-datetime repository, created 2019-10-17. https://github.com/Kotlin/kotlinx-datetime
[^2]: kotlinx-datetime README, "Design overview" — physical instant vs civil time, ISO 8601 scope, no i18n. https://github.com/Kotlin/kotlinx-datetime#design-overview
[^3]: kotlinx-datetime README — Alpha stability badge and Android core-library-desugaring requirement below API 26. https://github.com/Kotlin/kotlinx-datetime#using-in-your-projects
[^4]: kotlinx-datetime README, "Implementation" — JVM `java.time`, other platforms ThreeTen backport, JS/Wasm tz via js-joda. https://github.com/Kotlin/kotlinx-datetime#implementation
[^5]: kotlinx-datetime README, "Deprecation of `Instant`" — 0.7.x migration, compat artifact, kotlinx.serialization 1.9.0. https://github.com/Kotlin/kotlinx-datetime#deprecation-of-instant
[^6]: kotlinx-datetime README — compatibility with Kotlin Standard Library `2.3.21`; Maven Central artifact `0.8.0`. https://search.maven.org/search?q=g:org.jetbrains.kotlinx%20AND%20a:kotlinx-datetime

## Tags

kotlin, multiplatform, datetime, date, time, timezone, iso-8601, kotlin-multiplatform, jetbrains, library
