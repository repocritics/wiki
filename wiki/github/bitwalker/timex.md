# bitwalker/timex

> The kitchen-sink date/time library for Elixir — written before the standard
> library could do dates at all, and now partially superseded by it.

[GitHub repo](https://github.com/bitwalker/timex) ·
[Official docs](https://hexdocs.pm/timex) ·
[License: MIT](https://github.com/bitwalker/timex/blob/main/LICENSE.md)

## Overview

Timex is a date/time library for Elixir by Paul Schoenfelder (bitwalker, also
the author of distillery and libcluster). It started in 2014, when Elixir had
no `Date`, `Time`, `NaiveDateTime`, or `DateTime` types at all — those arrived
in Elixir 1.3 (2016). For years Timex was the de facto answer to "how do I do
dates in Elixir": timezone conversion via the IANA/Olson database (through the
`tzdata` package), strftime and Timex-native formatting, parsing, shifting,
diffing, `Duration`, and `Interval` types.

The defining tension of the project is that the Elixir standard library has
steadily absorbed its use cases. Elixir 1.8 added the
`Calendar.TimeZoneDatabase` behaviour and `DateTime.shift_zone/3`; 1.11 added
`Calendar.strftime/3`; 1.17 (2024) added `Duration` and `Date.shift/2` /
`DateTime.shift/2`[^1]. The Timex README itself now opens by recommending you
evaluate whether the standard `Calendar` API is sufficient before adding the
dependency, and Timex 3.x delegates to the standard library where possible[^2].
That is unusually honest maintainer guidance, and it is correct: for new
projects Timex is a convenience layer, not a necessity.

The project is in maintenance mode rather than active development: the 3.7.x
line has been patch-only for years, and the last push was June 2025 with ~68
open issues at ~1.8k stars. It still works — the API is stable and the
underlying tzdata updates independently — but do not expect new features.

## Getting Started

```elixir
# mix.exs
defp deps do
  [{:timex, "~> 3.7"}]
end
```

```elixir
use Timex   # aliases Timex, Duration, Interval, Timezone, etc.

Timex.now("America/Chicago")
#=> #DateTime<2026-07-17 09:30:30.120-05:00 CDT America/Chicago>

Timex.shift(Timex.now(), hours: 2, minutes: 13)
Timex.format!(Timex.now(), "{ISO:Extended}")
Timex.format!(Timex.now(), "%FT%T%:z", :strftime)
Timex.parse!("2026-07-17T12:30:30+00:00", "{ISO:Extended}")

interval = Timex.Interval.new(from: ~D[2026-07-01], until: [days: 3])
~D[2026-07-02] in interval   #=> true
```

## Architecture / How It Works

Timex is built on two protocols. `Timex.Protocol` defines the core operations
(shift, diff, set, etc.) over `Date`, `NaiveDateTime`, `DateTime`, and Erlang
datetime tuples; everything else in the library derives from it, and you can
implement it for custom calendar types. `Timex.Comparable` does the same for
comparisons. Formatting and parsing are pluggable behaviours
(`Timex.Format.DateTime.Formatter`, `Timex.Parse.DateTime.Parser`) with two
built-in syntaxes: a Timex-native directive format (`{ISO:Extended}`,
`{relative}`) and classic strftime[^2].

Timezone support is delegated to lau/tzdata, a separate OTP application that
compiles the IANA database into ETS tables and — by default — periodically
fetches updates from IANA over HTTP at runtime[^3]. Timex is oriented around
Olson/IANA zone names plus POSIX-TZ custom zones; support for bare timezone
abbreviations ("CST") existed in old versions, was broken by design (they are
ambiguous without context), and was removed.

Where an equivalent exists, Timex 3.x calls into the standard `Calendar` API
rather than reimplementing it, which keeps semantics aligned with stdlib but
also means much of the library is now a thin veneer[^2]. Distinctive leftover
value: relative formatting ("3 minutes ago"), `Interval` with `in`/overlap
semantics, `Duration` interop with Erlang timer tuples, ambiguous/nonexistent
local-time resolution via `AmbiguousDateTime`, and a large parsing toolkit.

## Production Notes

- **tzdata runs code and network calls you did not write.** The default
  auto-update fetches new IANA releases at runtime; in locked-down or
  airgapped environments this fails (or is a compliance surprise). Disable it
  with `config :tzdata, :autoupdate, :disabled` and ship updates via releases[^3].
- **First boot cost.** tzdata loads ETS tables at application start; cold
  boots and CI runs pay this even if you touch one timestamp.
- **escripts are a known trap.** Recent tzdata versions cannot load their ETS
  files from priv inside an escript; the README's workaround is pinning
  `{:tzdata, "~> 0.1.8", override: true}` — an ancient, stale database[^2].
  Avoid Timex in escripts.
- **You may not need it.** On Elixir >= 1.17, `DateTime.shift/2` + `Duration` +
  `Calendar.strftime/3` + `DateTime.shift_zone/3` (with tzdata or
  time_zone_info configured as the `:time_zone_database`) cover the common 90%
  without this dependency[^1].
- **Upgrade pain is historical but real.** Pre-3.x code (Timex 1.x/2.x `Date`
  and `DateTime` custom structs) required significant rewrites when 3.0 moved
  to stdlib types; APIs were renamed, removed, or changed semantics. Anything
  you inherit that predates 3.0 needs a careful migration pass[^2].
- **Companion packages are dead ends.** `timex_ecto` is unnecessary with
  modern Ecto, which handles `:utc_datetime`/`:naive_datetime` natively.
- **Maintenance reality.** Issues sit open for long periods; do not depend on
  upstream fixes landing quickly.

## When to Use / When Not

**Use when:**
- You maintain an existing codebase already built on Timex — it is stable and
  migration off is rarely worth it.
- You need relative time formatting, intervals, or the rich parser toolkit and
  do not want to hand-roll them.
- You support Elixir versions older than 1.17 and need duration arithmetic or
  shifting across DST boundaries.

**Avoid when:**
- You are starting a new project on Elixir >= 1.17 — stdlib `Calendar` +
  `DateTime.shift_zone/3` with a timezone database covers most needs[^1].
- You are building an escript.
- You only need timezone conversion — configure tzdata or time_zone_info as
  the `Calendar.TimeZoneDatabase` directly and skip the wrapper.
- Dependency weight matters: Timex pulls in tzdata, gettext, and combine.

## Alternatives

- elixir-lang/elixir — the stdlib `Calendar`/`Date`/`DateTime`/`Duration` API;
  use it instead for any new project on 1.17+ that does not need Timex's
  formatting/interval extras.
- lau/tzdata — use directly (as a `Calendar.TimeZoneDatabase`) when all you
  need is timezone conversion.
- hrzndhrn/time_zone_info — alternative timezone database; no runtime HTTP
  auto-update daemon, works better in restricted environments.
- elixir-cldr/cldr_dates_times — use instead when you need properly localized
  date/time formatting (CLDR locales) rather than strftime.
- lau/calendar — Timex's historical contemporary; superseded by stdlib.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014-07 | Project created; predates Elixir's built-in calendar types. |
| 3.0 | 2016 | Major rework onto Elixir 1.3's new stdlib `Date`/`DateTime` types; breaking API changes[^2]. |
| 3.7 | 2020 | Current release line; delegates to stdlib where possible, requires Elixir 1.8+. |
| 3.7.x | 2021–2025 | Patch-only maintenance; last push 2025-06. |

## References

[^1]: Elixir team, "Elixir v1.17 released" (Duration, `Date.shift/2`, `DateTime.shift/2`) — 2024-06-12. https://elixir-lang.org/blog/2024/06/12/elixir-v1-17-0-released/
[^2]: Timex README, "Migrating to Timex 3.x" and escript notes. https://github.com/bitwalker/timex#migrating-to-timex-3x
[^3]: Tzdata, "Automatic data updates". https://github.com/lau/tzdata#automatic-data-updates

## Tags

elixir, datetime, timezone, calendar, date-parsing, date-formatting, duration, intervals, tzdata, library
