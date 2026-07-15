# JodaOrg/joda-time

> The de facto Java date-time library of the pre-Java-8 era — now in maintenance-only mode, superseded by its own descendant `java.time`.

[GitHub repo](https://github.com/JodaOrg/joda-time) ·
[Official website](https://www.joda.org/joda-time/) ·
[License: Apache-2.0](https://www.joda.org/joda-time/licenses.html)

## Overview

Joda-Time is a date and time library for Java, created by Stephen Colebourne to
replace the deeply flawed `java.util.Date` and `java.util.Calendar` classes that
shipped with the JDK. Its core design ideas — immutable value types, a clean
separation between instants, partials (like `LocalDate`), durations, and periods,
and pluggable chronologies (ISO, Gregorian, Julian, Buddhist, Coptic, Islamic,
and others) — became the reference model for how date-time handling *should* work
in a typed language.

The defining fact about Joda-Time in 2026 is that it is explicitly finished. The
README states it "is no longer in active development except to keep timezone data
up to date"[^1]. This is not abandonment but a rare, deliberate wind-down: the
same author, Stephen Colebourne, went on to lead JSR-310, which brought
`java.time` into Java SE 8 in 2014[^2]. `java.time` is essentially Joda-Time's
successor, redesigned with the benefit of hindsight (nanosecond precision, a
cleaner nullable story, `Instant`/`ZonedDateTime`/`LocalDate` split). The
project's own maintainers ask Java 8+ users to migrate.

So the tension here is unusual: Joda-Time is high quality, battle-tested, and
still pulled by an enormous swath of the Maven ecosystem, yet using it in new
code is a documented mistake if you are on a modern JDK. It survives mostly as a
transitive dependency and in legacy codebases that have not paid down the
migration.

## Getting Started

Maven:

```xml
<dependency>
  <groupId>joda-time</groupId>
  <artifactId>joda-time</artifactId>
  <version>2.14.2</version>
</dependency>
```

Gradle:

```groovy
implementation 'joda-time:joda-time:2.14.2'
```

Minimal example:

```java
import org.joda.time.*;

LocalDate today = LocalDate.now();
LocalDate newYear = today.plusYears(1).withDayOfYear(1);
Days remaining = Days.daysBetween(today, newYear);

DateTime now = new DateTime(DateTimeZone.forID("Europe/London"));
Period rental = new Period().withDays(2).withHours(12);
boolean overdue = now.plus(rental).isBeforeNow();
```

Note the classic footgun preserved from the API: months are 1-based
(`getMonthOfYear() == 2` is February), which is more sane than
`java.util.Calendar`'s 0-based months but differs from some ports.

## Architecture / How It Works

Joda-Time separates the domain into a small number of orthogonal concepts:

- **Instants** — `DateTime`, `Instant`, `DateMidnight` (deprecated): a specific
  point on the timeline, tied to a chronology and zone.
- **Partials** — `LocalDate`, `LocalTime`, `LocalDateTime`, `YearMonth`,
  `MonthDay`: a date/time with no zone, representing "some 3 PM" rather than a
  specific instant.
- **Durations vs Periods** — a `Duration` is an exact millisecond span; a
  `Period` is a human span ("2 months, 3 days") whose millisecond length depends
  on when it is applied. Conflating these is the most common date-math bug and
  Joda-Time was one of the first libraries to force the distinction into the type
  system.
- **Chronology** — the pluggable calendar-system engine. `ISOChronology` is the
  default; every field operation (`monthOfYear()`, `dayOfWeek()`) routes through
  the active chronology, which is how alternate calendars are supported without
  branching in user code.

Almost all core types are immutable and thread-safe; mutation methods return new
instances (`withDayOfYear`, `plusDays`). This immutability was a deliberate
reaction to the mutable, footgun-heavy `java.util.Calendar`.

Time-zone data comes from the IANA TZDB, compiled into the jar. This is the one
part of the library that still changes: point releases (2.x) exist almost
entirely to ship updated zone rules as governments alter DST and offsets[^1].

## Production Notes

**The timezone-data coupling is the real operational concern.** Because zone
rules are baked into the jar, an old Joda-Time pin will silently compute wrong
local times after a country changes its DST policy. Keep the dependency current
even though the code is "finished" — the whole point of ongoing releases is the
TZDB refresh. External zone data can be supplied via
`DateTimeZone.setProvider`, but almost nobody does.

**Migration to `java.time` is the dominant long-term cost.** The APIs look
similar but are not drop-in: Joda's `DateTime` maps loosely to
`java.time.ZonedDateTime`, `LocalDate`/`LocalTime`/`LocalDateTime` names collide
but have subtly different semantics, and Joda `Period`/`Duration` do not line up
one-to-one with `java.time.Period`/`Duration`. Teams typically bridge at
serialization boundaries (Jackson has both `jackson-datatype-joda` and
`jackson-datatype-jsr310`) and migrate module by module.

**Null and parsing behavior differ from the JDK.** Joda-Time formatters
(`DateTimeFormat.forPattern`) are lenient in places `java.time`'s
`DateTimeFormatter` is strict, so a mechanical port can change which inputs parse.

**Android:** for `minSdk` below 26, `java.time` is unavailable natively; the
common replacement is not Joda-Time but ThreeTenABP, a backport of JSR-310, which
the README itself recommends[^1]. There was also a stripped-down `joda-time-android`
fork built for method-count-sensitive Android apps, but ThreeTenABP is now the
standard answer.

**Security/support:** vulnerability handling is routed through Tidelift rather
than public GitHub issues[^1].

## When to Use / When Not

**Use when:**
- You are on Java 6/7 and cannot use `java.time` at all.
- You are maintaining an existing Joda-Time codebase and a full migration is not
  yet justified — keep the dependency patched for TZDB updates.
- A dependency you rely on exposes Joda types in its public API.

**Avoid when:**
- You are writing new code on Java 8 or later — use `java.time` (JSR-310), which
  is the maintainer-endorsed successor.
- You need nanosecond precision (Joda-Time is millisecond-resolution).
- You are on modern Android (`minSdk` 26+) — use `java.time` directly; below that,
  use ThreeTenABP.

## Alternatives

- java.time (JSR-310, part of the JDK) — the successor by the same author; use it for any new code on Java 8+.
- ThreeTen/threetenbp — Stephen Colebourne's standalone backport of `java.time` for Java 6/7; use when you want JSR-310 semantics on an old JDK.
- JakeWharton/ThreeTenABP — Android packaging of threetenbp with lazy TZDB loading; use for Android `minSdk` < 26.
- apache/commons-lang (DateUtils) — use only for small utility helpers over legacy `java.util.Date`, not as a full date model.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2005-01 | First stable release; immutable model, pluggable chronologies. |
| 2.0 | 2011-08 | Major release; `java.util.Date`-free internals, improved partials. |
| — | 2014-03 | Java SE 8 ships `java.time` (JSR-310), the maintainer-authored successor[^2]. |
| 2.9.x | 2015–2017 | Steady TZDB refresh releases; API considered complete. |
| 2.10.x | 2018–2020 | Continued zone-data updates; project declared maintenance-only. |
| 2.14.2 | 2026 | Current release; JDK 1.5+, stable, TZDB updates only[^1]. |

## References

[^1]: Joda-Time README and project home page — migration guidance, maintenance
status, TZDB update policy, Android/ThreeTenABP recommendation, Tidelift security
contact. https://www.joda.org/joda-time/
[^2]: JSR-310 "Date and Time API", delivered in Java SE 8 (2014), led by Stephen
Colebourne — the maintainer-endorsed successor to Joda-Time.
https://jcp.org/en/jsr/detail?id=310

## Tags

java, date-time, jvm, immutable, timezone, calendar-systems, jsr-310, legacy, maintenance-mode, library
