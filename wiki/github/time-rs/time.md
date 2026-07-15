# time-rs/time

> Date and time handling in Rust — a `no_std`-capable library built around fixed UTC offsets, with no bundled IANA timezone database.

[GitHub repo](https://github.com/time-rs/time) ·
[Official website](https://time-rs.github.io) ·
[License: MIT OR Apache-2.0](https://github.com/time-rs/time/blob/main/LICENSE-Apache)

## Overview

`time` is one of the oldest crates in the Rust ecosystem — the repository dates to 2014, before Rust 1.0[^1]. The original `time 0.1` was a thin binding over the platform's C time functions; the more ergonomic `chrono` crate grew up alongside it and, for years, was the default recommendation. The `time` crate seen today is effectively a different library: a ground-up rewrite led by Jacob Pratt (`jhpratt`) as `0.2` in 2020 and stabilized into the current `0.3` line in 2021[^2]. Development is still overwhelmingly single-maintainer.

The defining design choice is that `time` models **only fixed UTC offsets**, not named timezones. There is no bundled IANA/Olson database, so `time` cannot tell you that `America/New_York` was `-05:00` in January and `-04:00` in July. `OffsetDateTime` carries a `UtcOffset` (a fixed `±HH:MM:SS`), and that is the extent of timezone awareness. This keeps the crate small, `no_std`-friendly, and free of a multi-megabyte tz database — at the cost of being unable to do civil-time arithmetic across DST transitions without an external timezone source.

The second thing worth knowing before adopting it: `time::Duration` is **signed** and distinct from `std::time::Duration` (unsigned). Mixing the two is a frequent source of confusion, and the API deliberately does not implicitly convert between them.

## Getting Started

```toml
# Cargo.toml
[dependencies]
time = { version = "0.3", features = ["macros", "formatting", "parsing"] }
```

```rust
use time::{Duration, OffsetDateTime};
use time::macros::datetime;
use time::format_description::well_known::Rfc3339;

fn main() -> Result<(), time::Error> {
    // Current instant in UTC (requires the default `std` feature).
    let now = OffsetDateTime::now_utc();
    let deadline = now + Duration::hours(48);

    // Compile-time-checked literal via the `macros` feature.
    let launch = datetime!(2026-07-15 09:30 UTC);

    println!("deadline: {}", deadline.format(&Rfc3339)?);
    println!("days until launch: {}", (launch - now).whole_days());
    Ok(())
}
```

Almost everything is gated behind feature flags. A bare `time = "0.3"` gives you the types and arithmetic but **not** formatting, parsing, or the `date!`/`datetime!`/`format_description!` macros — a common first-time surprise.

## Architecture / How It Works

The core types are plain value structs, each validated on construction:

- **`Date`** — a calendar date (proleptic Gregorian). Range is `±9999` years by default; the `large-dates` feature extends this to `±999_999`.
- **`Time`** — time-of-day with nanosecond precision, no date attached.
- **`PrimitiveDateTime`** — `Date` + `Time` with no offset (naive/civil time).
- **`OffsetDateTime`** — a `PrimitiveDateTime` plus a fixed `UtcOffset`.
- **`UtcOffset`** — a fixed `±HH:MM:SS` offset. Not a timezone.
- **`Duration`** — a signed span (seconds + nanoseconds). Distinct from `std::time::Duration`.
- Support types: `Month`, `Weekday`, `Instant` (a monotonic wrapper, `std` only).

**Formatting and parsing** are driven by *format descriptions* rather than `strftime`-style strings. You can build one at compile time with `format_description!`, parse one at runtime, or use the `well_known` set: `Rfc3339`, `Rfc2822`, and `Iso8601`. This is more verbose than `strftime` but type-checked and unambiguous[^3].

**Feature flags** are the crate's real interface surface. Notable ones: `std` (default), `alloc`, `formatting`, `parsing`, `macros`, `serde`, `serde-well-known`, `local-offset`, `large-dates`, `rand`, `quickcheck`, and `wasm-bindgen`. `no_std` builds drop `std` and opt into `alloc` (or neither) — the crate is usable on embedded targets, which is a large part of why it exists as an alternative to `chrono`.

**Local time** is the sharp edge. Determining the machine's local offset requires the `local-offset` feature, and `UtcOffset::local_offset_at` / `OffsetDateTime::now_local` return a `Result` that **fails in a multi-threaded process on Unix**. This is not a bug — see Production Notes.

## Production Notes

**The `local-offset` soundness restriction.** `time` historically called C's `localtime_r`, which reads the `TZ` environment variable. Because Rust's `std::env::set_var` is not synchronized against libc's environment access, a concurrent `setenv` from another thread could cause a segfault — filed as RUSTSEC-2020-0071 and affecting `chrono` for the same reason[^4]. `time`'s fix was to refuse to compute the local offset unless it can prove the process is effectively single-threaded at that moment. In practice this means `OffsetDateTime::now_local()` returns `Err(IndeterminateOffset)` inside most servers, async runtimes, and test harnesses. Workarounds: capture the offset once at startup before spawning threads, set `RUSTFLAGS="--cfg unsound_local_offset"` to opt back into the old behavior (not recommended), or obtain the offset from an external source (e.g. the `tz`/`jiff` ecosystem).

**No timezone database.** Any workload that needs "9 AM in Tokyo next Tuesday" or DST-correct arithmetic must supply offsets from elsewhere. `time` pairs with `tz-rs`/`tzdb` if you need this; if named-zone civil arithmetic is central to your app, `time` is the wrong base layer.

**Feature-flag friction.** Because formatting, parsing, macros, and serde are all opt-in, dependency trees frequently under-enable them, producing "method not found" errors that are really missing features. Enabling `formatting`/`parsing` pulls in more code and compile time; `no_std` users trade convenience for size.

**Serde behavior.** The plain `serde` feature serializes to `time`'s own representation. For interop you almost always want `serde-well-known` (RFC 3339 strings) and, for compact binary/human-readable duality, the `serde::rfc3339`/`format_description` module attributes on your fields. Getting this wrong yields technically-valid but ugly or non-interoperable output.

**Signed vs unsigned durations.** Converting to/from `std::time::Duration` is explicit and fallible (a negative `time::Duration` has no `std` equivalent). Interop with APIs that speak `std::time::Duration` (most of async Rust) requires deliberate conversion.

**MSRV policy.** `time` supports the latest stable Rust plus the two prior minor releases (currently 1.88.0), and reserves the right to bump the floor to four minors back for maintainability. This is a relatively aggressive MSRV compared to libraries that pin years behind stable — pinning `time` in a conservative toolchain environment can force a Rust upgrade.

## When to Use / When Not

**Use when:**
- You need dates/times in `no_std` or embedded contexts where `chrono` or a tz database won't fit.
- Your domain is UTC or fixed-offset (logs, timestamps, wire formats, RFC 3339 I/O).
- You want compile-time-validated datetime literals and format descriptions.
- You want a small dependency with an explicit, feature-gated surface.

**Avoid when:**
- You need named-timezone / DST-aware civil arithmetic out of the box — reach for a tz-database-backed crate.
- You want `strftime`-style format strings and a batteries-included default (`chrono` is friendlier here).
- You need reliable local-time detection inside a multithreaded server without extra plumbing.

## Alternatives

- chronotope/chrono — the long-time default; larger API, `strftime` formatting, optional `chrono-tz` for named zones. Use instead when you want the more established, batteries-included option.
- BurntSushi/jiff — newer datetime library with a bundled IANA tz database and DST-correct arithmetic. Use instead when named-timezone civil time is central.
- x-hgg-x/tz-rs — pure-Rust IANA tz parser, `no_std`-capable. Use alongside `time`/others to supply real timezone offsets.
- std::time — `SystemTime`/`Instant`/`Duration` only. Use instead when you need nothing more than monotonic clocks and wall-clock instants.
- time-rs/time 0.1 (same repo, legacy) — do not adopt for new code; the 0.3 line supersedes it entirely.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2015–2019 | Original thin libc binding; one of Rust's oldest crates[^1]. |
| 0.2.0 | 2020-01 | Full rewrite led by Jacob Pratt; new type model[^2]. |
| 0.2.23 | 2020-11 | Mitigation for RUSTSEC-2020-0071 (local-offset segfault)[^4]. |
| 0.3.0 | 2021-07 | Current API line; feature-gated formatting/parsing, `no_std` refinements. |
| 0.3.53 | 2026-07-01 | Latest 0.3 patch release; MSRV 1.88.0[^5]. |

No 1.0 release has shipped as of mid-2026; the crate has lived on the `0.3` line for years while remaining widely used across the ecosystem. As of this writing it sits at ~1.3k stars and ~300 forks, with releases still landing regularly.

## References

[^1]: `time-rs/time` repository, created 2014-11-10. https://github.com/time-rs/time
[^2]: Crate metadata and changelog, `time` 0.2 rewrite. https://github.com/time-rs/time/blob/main/CHANGELOG.md
[^3]: `time` format description documentation. https://time-rs.github.io/book/api/format-description.html
[^4]: RUSTSEC-2020-0071, "Potential segfault in the time crate." https://rustsec.org/advisories/RUSTSEC-2020-0071.html
[^5]: Release `v0.3.53`, published 2026-07-01. https://github.com/time-rs/time/releases/tag/v0.3.53

## Tags

rust, date-time, no-std, datetime, timezone, utc, serde, embedded, timestamp, formatting
