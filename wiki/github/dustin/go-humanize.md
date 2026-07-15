# dustin/go-humanize

> Formatters that turn machine numbers — byte counts, timestamps, big integers — into strings a human reads without thinking.

[GitHub repo](https://github.com/dustin/go-humanize) ·
[Docs (pkg.go.dev)](https://pkg.go.dev/github.com/dustin/go-humanize) ·
License: MIT (GitHub reports NOASSERTION)[^1]

## Overview

go-humanize is a small Go library, started in 2012 by Dustin Sallings, that
converts raw values into human-friendly strings: `82854982` becomes `83 MB`,
a `time.Time` becomes `7 hours ago`, `6582491` becomes `6,582,491`, and `193`
becomes `193rd`[^2]. It is the de facto standard for this task in the Go
ecosystem and appears transitively in a large share of Go binaries — CLI
tools, dashboards, and log formatters lean on it rather than reinventing the
comma-and-suffix logic.

The library is deliberately narrow and effectively feature-complete. It has no
third-party dependencies (standard library only) and the public API has been
stable for years; commits are dominated by edge-case fixes and doc tweaks
rather than new surface. In practical terms this is "done" software: you pull
it in, it does one job, and it does not churn under you.

The defining tension is scope. go-humanize is English-only and locale-blind by
design — output strings, pluralization rules, and relative-time phrasing are
hardcoded for English[^3]. That is exactly why it is small and dependency-free,
and exactly why it is the wrong tool the moment you need i18n. It solves the
developer-facing-English case and explicitly declines localization.

## Getting Started

```bash
go get github.com/dustin/go-humanize
```

```go
package main

import (
	"fmt"
	"time"

	"github.com/dustin/go-humanize"
)

func main() {
	fmt.Println(humanize.Bytes(82854982))   // 83 MB   (SI, base 1000)
	fmt.Println(humanize.IBytes(82854982))  // 79 MiB  (IEC, base 1024)
	fmt.Println(humanize.Comma(6582491))     // 6,582,491
	fmt.Println(humanize.Ordinal(193))       // 193rd
	fmt.Println(humanize.Ftoa(2.240))        // 2.24    (trailing zeros dropped)
	fmt.Println(humanize.Time(time.Now().Add(-3 * time.Hour))) // 3 hours ago
}
```

## Architecture / How It Works

go-humanize is a flat package of pure functions plus one subpackage; there is
no state, no config object, no initialization. The interesting design choices
are in the number/unit logic:

- **Byte sizes come in two families.** `Bytes`/`Format` use SI decimal units
  (kB, MB, GB — powers of 1000). `IBytes`/`FormatIEC` use IEC binary units
  (KiB, MiB, GiB — powers of 1024)[^2]. Both exist because both conventions are
  in real-world use; the library refuses to pick one for you.
- **Big-number variants** (`BigBytes`, `BigComma`, `BigIBytes`) accept
  `math/big.Int` so values beyond `uint64`/`int64` still format correctly.
- **`Comma`/`Commaf`/`CommafWithDigits`** insert thousands separators; the
  float variants exist separately because Go has no function overloading.
- **`Ftoa`/`FtoaWithDigits`** wrap `strconv.FormatFloat` and strip trailing
  zeros, producing `2` instead of `2.000000`.
- **`Time`** computes a relative phrase against `time.Now()`. Under it,
  `RelTime(a, b, albl, blbl)` is the general form that takes both endpoints and
  the "ago"/"from now" labels, so you can supply a fixed reference time — the
  seam you need for deterministic tests.
- **`SI`/`ParseSI`** apply and parse metric prefixes (`2.23 nM`), and
  `ParseBytes` reverses `Bytes` back into a `uint64`.
- **`humanize/english` subpackage** handles pluralization (`PluralWord`,
  `Plural`) and joining word lists with conjunctions (`WordSeries`,
  `OxfordWordSeries`). Its plural rules are heuristic, not a full morphology
  engine.

Because every entry point is a pure function over primitives, the library is
trivially safe for concurrent use and has no lifecycle to manage.

## Production Notes

The footguns are all about which function you called, not about failure modes:

- **SI vs IEC is the classic bug.** `Bytes(1048576)` is `1.0 MB`; `IBytes`
  of the same value is `1.0 MiB`. Teams that want "1 MiB" but call `Bytes` ship
  numbers that disagree with `df`, `du`, and every OS file dialog. Decide the
  convention once and grep for the wrong function in review.
- **`Time` is non-deterministic by construction.** It reads the wall clock, so
  snapshot tests and golden files break. Use `RelTime` with an injected
  reference time in anything you assert on.
- **Relative time is coarse.** `Time` buckets into human magnitudes ("about an
  hour ago", "2 days ago") and rounds hard. It is presentation, not a duration
  you can reverse into an exact interval.
- **No localization, and no hook to add it.** Strings are compiled in. If a
  product later needs non-English output, this is a rip-and-replace, not a
  configuration change — plan the abstraction boundary early if i18n is even
  possible on the roadmap.
- **Pluralization is best-effort.** `english.PluralWord` handles common English
  patterns but is not a dictionary; irregular or domain-specific plurals need
  the explicit-override argument (`PluralWord(99, "locus", "loci")`).
- **Versioning is quiet.** The package moves slowly and rarely tags releases,
  so `go get` without a version pin can sit on a commit rather than a tag.
  Pin to `v1.0.1` (or the current tag) in `go.mod` for reproducibility.

## When to Use / When Not

**Use when:**
- You are formatting numbers, byte counts, or timestamps for a developer- or
  admin-facing English UI, CLI, or log line.
- You want zero dependencies and a stable API you will not have to babysit.
- You need SI/IEC byte formatting or comma grouping and don't want to hand-roll
  the edge cases (negatives, big.Int, rounding boundaries).

**Avoid when:**
- You need localized or multi-language output — reach for
  `golang.org/x/text/message` and friends instead.
- You need exact, reversible durations rather than fuzzy "3 days ago" phrasing.
- Your pluralization needs are linguistically serious (large vocabularies,
  irregular forms) — a dedicated inflection library fits better.

## Alternatives

- golang.org/x/text — the locale-aware answer; `message.Printer` formats
  numbers, plurals, and currency per language tag. Use it when i18n is a
  requirement, accepting far more surface area.
- docker/go-units — Docker's own byte/duration formatters (`HumanSize`,
  `BytesSize`); use it when you're already in that ecosystem or want just the
  size helpers.
- gertd/go-pluralize — a fuller English pluralization/singularization engine;
  use it when `humanize/english`'s heuristics are too thin.
- c2h5oh/datasize — a typed `ByteSize` with parsing and arithmetic; use it when
  you want byte sizes as first-class values, not just formatted strings.
- inhies/go-bytesize — another typed byte-size package with parse/format; use
  it as a lighter datasize alternative.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2012-01-13 | Repository created by Dustin Sallings[^1]. |
| v1.0.0 | 2018-03-09 | First tagged semantic-version release. |
| v1.0.1 | 2019-12-30 | Latest tagged release; subsequent work is edge-case fixes. |

## References

[^1]: GitHub API — repository metadata for `dustin/go-humanize`, retrieved
2026-07-15: 4,809 stars, 254 forks, 26 watchers, 56 open issues, created
2012-01-13, last push 2026-03-02, default branch `master`. License field
returns `NOASSERTION` although the repository ships an MIT license text.
https://github.com/dustin/go-humanize/blob/master/LICENSE
[^2]: Package README and API examples. https://github.com/dustin/go-humanize
[^3]: `humanize/english` subpackage documentation.
https://pkg.go.dev/github.com/dustin/go-humanize/english

## Tags

go, golang, formatting, humanize, byte-size, number-formatting, relative-time, pluralization, cli, i18n, utility
