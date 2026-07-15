# python-pendulum/pendulum

> A datetime library for Python that subclasses the standard `datetime` and makes every instance timezone-aware by default.

[GitHub repo](https://github.com/python-pendulum/pendulum) ·
[Official website](https://pendulum.eustace.io) ·
[License: MIT](https://github.com/python-pendulum/pendulum/blob/master/LICENSE)

## Overview

Pendulum is a datetime library written by Sébastien Eustace (`sdispater`, also the author of Poetry) and first released in 2016[^1]. Its central design decision is that `pendulum.DateTime` inherits directly from the standard library's `datetime.datetime` (and `Date` from `date`, `Duration` from `timedelta`). This makes it a near drop-in replacement: existing code that expects a `datetime` continues to work, while gaining a cleaner API for arithmetic (`.add()`, `.subtract()`), timezone shifting (`.in_timezone()`), human-friendly diffs (`.diff_for_humans()`), and localization.

The library takes an opinionated stance the standard library does not: it removes the concept of a naive datetime. Every Pendulum instance is timezone-aware, defaulting to UTC. This eliminates the most common category of Python datetime bug — mixing naive and aware objects and getting `TypeError` or silently wrong arithmetic — at the cost of forcing a timezone decision on you up front.

That subclassing choice is also Pendulum's defining tension. Because a `DateTime` *is* a `datetime`, most of the ecosystem accepts it transparently, but any library that inspects the concrete type with `type(obj)` rather than `isinstance()` — historically `sqlite3`, `PyMySQL`/`mysqlclient`, and Django's ORM adapters — will not recognize it and needs a registered adapter[^2]. Since Python 3.9 shipped `zoneinfo` (PEP 615) in the standard library, the original "you need this for sane timezones" justification has weakened, and Pendulum today competes more on ergonomics than necessity.

## Getting Started

```bash
pip install pendulum
```

```python
import pendulum

now = pendulum.now("Europe/Paris")
now.in_timezone("UTC")              # seamless tz conversion

tomorrow  = pendulum.now().add(days=1)
last_week = pendulum.now().subtract(weeks=1)

past = pendulum.now().subtract(minutes=2)
past.diff_for_humans()              # '2 minutes ago'

# Durations and human-readable spans
delta = tomorrow - last_week
delta.in_words(locale="en")         # e.g. '8 days'

# DST is handled correctly — 2:30 does not exist on this spring-forward day
pendulum.datetime(2013, 3, 31, 2, 30, tz="Europe/Paris")
# -> 2013-03-31T03:30:00+02:00
```

## Architecture / How It Works

Pendulum is a thin, opinionated layer built *on top of* the standard datetime types rather than a reimplementation:

- **`DateTime`** subclasses `datetime.datetime`; **`Date`** subclasses `date`; **`Time`** subclasses `time`; **`Duration`** subclasses `timedelta`. Because they are subclasses, `isinstance(x, datetime)` is always true, and stdlib functions that accept a `datetime` accept a `DateTime`.
- **`Interval`** represents the span between two datetimes (start/end aware). In Pendulum 3.0 this class was renamed from `Period`, which was a long-standing source of confusion versus `Duration` (a bare length of time with no anchor)[^3].
- **Timezones** are handled by Pendulum's own `Timezone` implementation with explicit rules for DST transitions: skipped ("spring forward") times are normalized forward, and ambiguous ("fall back") times resolve deterministically rather than raising.
- **Parsing** in Pendulum 3.0 is backed by a compiled Rust extension for RFC 3339 / ISO 8601 strings, replacing the pure-Python parser used through the 2.x line[^3]. `pendulum.parse()` handles ISO 8601 dates, times, durations, and intervals.
- **Localization** ships CLDR-derived locale data under `pendulum/locales/`, driving `diff_for_humans()` and `format()` output. Each locale has a machine-generated `locale.py` (never edited) and a hand-maintained `custom.py`.

The immutability model matters: Pendulum instances are immutable like stdlib `datetime`, so `.add()` / `.subtract()` return new objects rather than mutating in place.

## Production Notes

- **`type()`-based adapters break.** The subclassing trick fails for any driver that keys on the exact class. `sqlite3`, `PyMySQL`, and `mysqlclient` all require registering an adapter that serializes `DateTime` (e.g. via `isoformat()`), and Django's MySQL backend can raise on the always-present offset. The README documents concrete workarounds for each[^2]. Test your persistence layer specifically; the failure is at write time, not import time.
- **The stdlib caught up.** For pure timezone correctness, `datetime` + `zoneinfo` (3.9+) now covers what once required Pendulum. Adopt Pendulum for the API (arithmetic, humanized diffs, parsing, i18n), not because the standard library can't do timezones.
- **Compiled extension.** Since 3.0, installing Pendulum may pull a platform wheel with the Rust parser. Wheels exist for common platforms, but exotic architectures or locked-down build environments can fall back to building from source (needs a Rust toolchain). Pin versions in constrained CI.
- **Python version churn.** The current line supports Python 3.10 and newer; the 3.0 release dropped older interpreters that 2.x still supported. Projects on legacy Python are stuck on the 2.x branch, which no longer receives feature work.
- **Maintenance cadence is slow and bursty.** The project moved from the `sdispater` personal namespace to the `python-pendulum` organization, and its lead author's attention is largely on Poetry. There were multiple years between the 2.x series and the 3.0 release, and the open-issue backlog is large. It is maintained, but do not expect rapid turnaround on issues or fast support for brand-new interpreter releases.
- **`diff_for_humans()` is locale- and now-dependent.** Output depends on the current time and the active locale; freeze the clock (e.g. `pendulum.travel_to()` / `time-machine`) in tests rather than asserting against wall-clock-relative strings.

## When to Use / When Not

**Use when:**
- You want ergonomic datetime arithmetic and humanized diffs without writing helpers.
- Timezone-aware-by-default is a feature you want enforced across a codebase.
- You need built-in localization for human-readable time spans.
- You want objects that still pass `isinstance(x, datetime)` checks throughout the ecosystem.

**Avoid when:**
- Your only need is timezone support — `datetime` + `zoneinfo` has no dependency and no compiled extension.
- You rely on drivers/ORMs that inspect concrete types and you'd rather not maintain adapters.
- You want compile-time separation of aware vs. naive datetimes — a strictly-typed library like `whenever` enforces that; Pendulum does not.
- You need a library on an aggressive release cadence tracking the newest CPython.

## Alternatives

- arrow-py/arrow — wraps `datetime` instead of subclassing it; use when you prefer a self-contained wrapper type and don't need `isinstance(x, datetime)` to hold.
- ariebovenberg/whenever — Rust-backed, strictly typed, separates aware/naive/local at the type level; use when correctness guarantees and type safety matter more than a stdlib-compatible drop-in.
- dateutil/dateutil — flexible parsing plus `relativedelta`; use when your real need is fuzzy date parsing and calendar-relative deltas, not a datetime replacement.
- Standard library `datetime` + `zoneinfo` — use when you want zero dependencies and only need correct timezones.
- crsmithdev/arrow is the same project as arrow-py/arrow (renamed org); pick whichever wrapper-style API reads best to your team.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2016 | Initial releases; datetime-subclass design established[^1]. |
| 1.0 | 2016 | First stable API for `DateTime`, `Date`, arithmetic, humanized diffs. |
| 2.0 | 2018 | Major rewrite of internals and timezone handling. |
| 3.0 | 2024-02 | Rust parser extension, `Period` renamed to `Interval`, dropped older Python versions[^3]. |

## References

[^1]: Pendulum — repository created 2016-06-27, authored by Sébastien Eustace (`sdispater`). https://github.com/python-pendulum/pendulum
[^2]: Pendulum README, "Limitations" — documents `sqlite3`, `PyMySQL`/`mysqlclient`, and Django adapter workarounds for `type()`-based drivers. https://github.com/python-pendulum/pendulum/blob/master/README.rst
[^3]: Pendulum documentation and 3.0 release notes — Rust-based parsing and the `Period` → `Interval` rename. https://pendulum.eustace.io/docs/

## Tags

python, datetime, timezone, time, date, dst, localization, parsing, stdlib-replacement, developer-tools
