# chronotope/chrono

> The long-standing date and time library for Rust — timezone-aware `DateTime`, naive types, and `strftime`-style parsing, still pre-1.0 after a decade.

[GitHub repo](https://github.com/chronotope/chrono) ·
[API docs](https://docs.rs/chrono) ·
[License: MIT OR Apache-2.0](https://github.com/chronotope/chrono/blob/main/LICENSE.txt)

## Overview

Chrono is the oldest widely-used date and time library in the Rust ecosystem, with commit history reaching back to 2014[^1]. For most of Rust's existence it was the default answer to "how do I handle dates," and it remains one of the most-depended-on crates on crates.io. It models the [proleptic Gregorian calendar](https://en.wikipedia.org/wiki/Proleptic_Gregorian_calendar) with a timezone-aware `DateTime<Tz>` and a set of timezone-naive types (`NaiveDate`, `NaiveTime`, `NaiveDateTime`) for when offsets are irrelevant or supplied elsewhere[^2].

The defining tension in chrono is between its ubiquity and its unfinished API. Despite ~3,900 stars, hundreds of dependent crates, and continuous maintenance, it has never reached 1.0 — the entire public history lives in the `0.4.x` series[^3]. Because semver treats `0.4.x` as a single compatibility range, the maintainers have shipped deprecations and behavior refinements *within* patch releases rather than in a clean major bump. The result is a library that is stable in practice but littered with deprecated constructors and `_opt` variants, where the "correct" API to call has shifted several times without a version signal.

Chrono deliberately ships no timezone database. `DateTime` is offset-aware, but resolving a named zone like `America/New_York` requires the companion crate `chrono-tz` or `tzfile`; the `Local` timezone reads the OS setting at runtime[^2]. This keeps binaries small at the cost of making "what time is it in Tokyo" a two-crate problem.

## Getting Started

```toml
# Cargo.toml
[dependencies]
chrono = { version = "0.4", features = ["serde"] }
```

```rust
use chrono::{DateTime, Utc, NaiveDate, TimeZone};

fn main() {
    // Current time in UTC (requires the default `clock` feature).
    let now: DateTime<Utc> = Utc::now();
    println!("{}", now.to_rfc3339());

    // Fallible construction — invalid dates return None, not a panic.
    let d = NaiveDate::from_ymd_opt(2026, 2, 29); // not a leap year
    assert!(d.is_none());

    // Parsing with an strftime-style format.
    let parsed = Utc.datetime_from_str("2026-07-10 08:30:00", "%Y-%m-%d %H:%M:%S");
    assert!(parsed.is_ok());
}
```

The Minimum Supported Rust Version is 1.62.0, explicitly tested in CI and bumped only in minor releases[^4].

## Architecture / How It Works

Chrono splits its types along the axis of whether an instant is anchored to a timezone:

- **Naive types** (`NaiveDate`, `NaiveTime`, `NaiveDateTime`) carry no offset. They are cheap value types and the right choice for wall-clock data (a birthday, a business-hours schedule) where the zone is external.
- **`DateTime<Tz>`** pairs a naive instant with a timezone parameter implementing the `TimeZone` trait. `Utc` and `FixedOffset` are zero-cost; `Local` reads the OS zone; `chrono_tz::Tz` (external) provides IANA zones.

Operations that can be invalid or ambiguous do not panic in the modern API — they return `Option` or `MappedLocalTime` (formerly `LocalResult`), which encodes the three outcomes of mapping a local time to UTC: unambiguous, ambiguous (a fall-back DST overlap yields two candidates), or nonexistent (a spring-forward gap)[^2]. Correct DST-aware code must handle all three arms; a large share of chrono bugs in the wild come from `.unwrap()`-ing this away.

Feature flags gate the surface aggressively. `alloc` enables string formatting; `std` adds standard-library interop; `clock`/`now` enable reading system and local time; `serde` and the mutually-exclusive `rkyv-16/-32/-64` flags add serialization; `wasmbind` bridges to the JS `Date` API on `wasm32`. This makes chrono usable in `no_std` and embedded contexts, but means a dependency's chosen feature set can silently remove `Local::now` from your build.

A pivotal internal change came in **0.4.20** (2022): chrono removed its default dependency on the unmaintained `time` 0.1 crate and moved OS-zone detection to `iana-time-zone`, resolving the `localtime_r` soundness advisory (see Production Notes). The old path survives only behind the now-inert `oldtime` feature[^5].

## Production Notes

**Deprecation churn is the main operator tax.** Because everything is `0.4.x`, panicking constructors (`ymd`, `from_timestamp`, `and_hms`) were deprecated in favor of `_opt` and `_and_hms_opt` variants *inside* patch releases. Upgrading a patch version can flood a build with deprecation warnings even though nothing broke at runtime. Pin and read the changelog before bumping.

**Panicking vs. fallible APIs.** Older tutorials and Stack Overflow answers use the panicking constructors freely. On attacker-controlled or user-supplied input (`from_ymd(y, m, d)` with out-of-range values) these will panic. Prefer the `_opt` methods everywhere and treat any non-`_opt` date/time constructor in a code review as a latent panic.

**The `localtime_r` advisory (RUSTSEC-2020-0159).** Before 0.4.20, reading `Local::now()` could hit a segfault under concurrent `setenv`/`localtime_r` on some platforms, inherited from `time` 0.1[^6]. Projects on very old chrono pins are still exposed; upgrading to ≥ 0.4.20 is the fix. `cargo audit` will flag pre-0.4.20 versions.

**Timezone data is not included.** `chrono` alone cannot answer "convert this UTC instant to `Europe/Berlin`." You need `chrono-tz` (compiles the IANA database into the binary, adding size) or `tzfile` (reads the system zoneinfo at runtime). Teams routinely discover this only when a `Local`-based prototype fails to reproduce a specific zone in production.

**Leap seconds are representable but not arithmetic.** Chrono can store a leap-second instant (`:60`) but does not treat it correctly in duration math[^2]. Do not use chrono for anything requiring TAI/UTC leap-second accuracy.

**Range and precision limits.** Dates are bounded to roughly ±262,000 years from the epoch and times to nanosecond resolution[^2]. Sub-nanosecond or astronomical-scale needs are out of scope.

## When to Use / When Not

**Use when:**
- You want the ecosystem-default, maximally-compatible date library that nearly every other crate already speaks.
- You need `serde` round-tripping of RFC 3339 timestamps with minimal ceremony.
- You need `no_std`/`wasm` support and can select features to fit.
- You already depend on it transitively and want one consistent type across the tree.

**Avoid when:**
- You want a library that has committed to a stable 1.0 API — chrono's in-patch deprecations may frustrate you.
- You need first-class IANA timezone arithmetic and DST correctness without bolting on a second crate — evaluate `jiff`.
- You want a leaner, `const`-friendly API and can supply offsets yourself — evaluate `time`.
- You require leap-second-accurate or sub-nanosecond time.

## Alternatives

- time-rs/time — the primary alternative; smaller, `const`-friendly, `no_std`-first. Use instead when you don't need a bundled IANA database and prefer a cleaner, post-1.0-style API.
- BurntSushi/jiff — newer (2024), ships the IANA tzdb and is designed around DST/zone correctness. Use instead when timezone arithmetic correctness matters more than ecosystem ubiquity.
- chronotope/chrono-tz — companion, not competitor: adds IANA named-zone support on top of chrono. Use alongside chrono when `Utc`/`FixedOffset` isn't enough.
- unicode-org/icu4x — use instead when you need locale-aware calendars, non-Gregorian systems, or CLDR-backed formatting rather than plain timestamp handling.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014 | First commits; authored by Kang Seonghoon (lifthrasiir)[^1]. |
| 0.4.0 | 2017 | Start of the long-lived `0.4.x` series still in use today[^3]. |
| 0.4.19 | 2020 | Last release before the `time` 0.1 / `localtime_r` security work[^6]. |
| 0.4.20 | 2022-06 | Dropped default `time` 0.1 dep; moved to `iana-time-zone`; resolved RUSTSEC-2020-0159[^5]. |
| 0.4.23 | 2022-11 | Continued API stabilization and further deprecations of panicking constructors. |
| 0.4.x | 2024–2026 | Ongoing maintenance under the chronotope org; still no 1.0[^3]. |

## References

[^1]: chrono repository, created 2014-03-28. https://github.com/chronotope/chrono
[^2]: chrono README and API docs — types, `MappedLocalTime`, limitations, feature flags. https://docs.rs/chrono/latest/chrono/
[^3]: chrono on crates.io — version history within the `0.4.x` range. https://crates.io/crates/chrono/versions
[^4]: chrono README, "Rust version requirements" — MSRV 1.62.0. https://github.com/chronotope/chrono
[^5]: chrono 0.4.20 changelog — removal of `time` 0.1 dependency, `iana-time-zone` adoption. https://github.com/chronotope/chrono/blob/main/CHANGELOG.md
[^6]: RUSTSEC-2020-0159 — potential segfault in `localtime_r` invocations. https://rustsec.org/advisories/RUSTSEC-2020-0159.html

## Tags

rust, date-time, calendar, timezone, parsing, serde, no-std, wasm, library, gregorian
