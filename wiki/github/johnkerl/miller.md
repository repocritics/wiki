# johnkerl/miller

> awk, sed, cut, join, and sort for name-indexed data — CSV, TSV, JSON, and JSON Lines — as a single dependency-free binary.

[GitHub repo](https://github.com/johnkerl/miller) ·
[Official website](https://miller.readthedocs.io) ·
[License: BSD-2-Clause](https://github.com/johnkerl/miller/blob/main/LICENSE.txt)

## Overview

Miller (`mlr`) is a command-line tool for processing tabular data by field
name rather than by positional column index[^1]. The Unix classics operate on
whitespace- or delimiter-split fields addressed as `$1`, `$2`, `$3`; Miller
operates on records that are ordered key-value maps, so you write `$quantity`
instead of `$7`. Its natural data structure is the insertion-ordered hash map,
not the array. First published in 2015 as a C program, it was rewritten
end-to-end in Go for the 6.0 line and now ships as a single static binary with
zero runtime dependencies[^2].

The tool sits in a specific niche: too much for `cut`/`awk` one-liners, but
deliberately less than a database or a dataframe library. It is format-aware —
it understands CSV quoting, JSON nesting, and header rows as first-class
concepts, so a `sort` keeps the CSV header on top and a schema change emits a
new header block rather than corrupting the stream. It is also streaming by
default: most operations hold one record in memory at a time, which lets it run
on files larger than RAM and inside `tail -f` pipelines.

The defining tension is scope. Miller is a genuinely capable mini-language plus
~60 verbs, which means there are usually three ways to do anything and a real
learning curve past the trivial cases. For a quick CSV-to-JSON conversion it is
unbeatable; for a five-way relational join you are fighting the grain and
should reach for SQLite or DuckDB. Miller is maintained primarily by its
original author, John Kerl, and remains actively developed a decade on, with
commits landing regularly and roughly forty contributors credited.

## Getting Started

```bash
# Debian/Ubuntu / Fedora / macOS / Windows
apt-get install miller      # or: brew install miller / choco install miller
# from source (Go 1.x toolchain):
go install github.com/johnkerl/miller/v6/cmd/mlr@latest
```

```bash
# Convert CSV to pretty-printed table (--c2p == --icsv --opprint)
mlr --c2p cat example.csv

# Filter, derive a field, then aggregate — verbs chained with "then"
mlr --icsv --ojson \
  put '$profit = $revenue - $cost' \
  then filter '$profit > 0' \
  then stats1 -a sum,mean -f profit -g region \
  example.csv
```

The invocation shape is always `mlr [I/O flags] verb [verb args] then verb ...
[files]`. Format flags come first, verbs form a left-to-right pipeline, and the
file arguments come last (or stdin if omitted).

## Architecture / How It Works

Miller's execution model is a **record stream through a verb chain**. Each input
file is parsed into a sequence of records (ordered maps of string keys to typed
values); records flow one at a time through the verbs joined by `then`, and each
verb transforms, filters, or emits records downstream. This is the source of the
streaming property: `cat`, `cut`, `head`, `put`, and `filter` never buffer,
while a handful of verbs that inherently need the whole stream — `sort`, `tac`,
`stats1`/`stats2` accumulators — retain only what they must.

**I/O is decoupled from processing.** Input format and output format are set
independently (`--icsv --ojson`, `--itsv --opprint`, and so on), so Miller is
also a format converter as a side effect of doing anything at all. Supported
formats include CSV, TSV, JSON, JSON Lines, PPRINT (aligned tables), XTAB,
Markdown tables, NIDX (positionally-indexed, awk-style), and the default DKVP
(delimited key-value pairs, `a=1,b=2`). Shorthand flags like `--c2j` and `--j2c`
name common conversions.

**The DSL is a real language, not a flag.** The `put` and `filter` verbs embed
an interpreted mini-language with typed scalars, maps, arrays, control flow,
user-defined functions, `begin`/`end` blocks, out-of-stream variables (`@sum`,
persisting across records), and `emit` statements for building aggregate output.
Field references (`$x`), full-record access (`$*`), and oosvars (`@x`) are
distinct sigils. This is where Miller's power lives and where its learning curve
is steepest — the DSL overlaps with what verbs already do, so the same result
often has both a verb form and a `put` form.

**Record heterogeneity is a first-class property.** Because each record carries
its own key set, records with different schemas can be interleaved in one
stream. This is unremarkable in JSON but unusual for a CSV tool, and it shapes
output: heterogeneous records force new header lines in CSV and separate table
blocks in PPRINT.

## Production Notes

**`--csv` vs `--csvlite` is a real footgun.** The default CSV reader is
RFC-4180-compliant: it handles quoted fields, embedded commas, and embedded
newlines. `--csvlite` (and the `2` in some shorthands' lineage) is faster but
does *not* handle quoting — feeding it quoted data with embedded delimiters
silently mis-splits records. Know which one you invoked before trusting the
output on messy real-world data.

**Type inference can surprise.** Miller infers int vs float vs string per value
and preserves the distinction through to output, so `1` and `1.0` are not the
same on the way out, and octal-looking or leading-zero strings can be reinterpreted
as numbers. Use `-S` to treat all fields as strings when you need byte-exact
passthrough, and `--ofmt` to pin float formatting.

**Performance is "on par with the Unix toolkit," not beyond it.** For the
simplest column operations, `cut` and `awk` remain faster because they do less
parsing. Miller's advantage is format-awareness and streaming aggregation, not
raw throughput; for pure-speed CSV crunching on huge files, a Rust tool (xsv/xan)
or a columnar engine (DuckDB, Polars) will typically win. The C-to-Go rewrite in
v6 prioritized portability, correctness, and a cleaner DSL over microbenchmarks.

**Streaming has boundaries.** Verbs that must see all input — `sort`, `tac`,
`count-distinct`, grouped `stats` — accumulate state proportional to the data or
the group count. "Larger than RAM" holds only for the streaming verbs; a global
sort of an enormous file still needs the memory.

**Version pinning across distros.** Package repos lag, and LTS distro releases
often ship an older major version (the README flags this). DSL features and
shorthand flags have accreted over versions, so a script that works on your
laptop's Homebrew build may fail on a server's `apt` Miller — check
`mlr version` before assuming a feature exists.

## When to Use / When Not

**Use when:**
- You need name-based field manipulation and format conversion among CSV, TSV,
  JSON, and JSON Lines without writing a script.
- You want streaming, single-pass aggregation over data larger than memory.
- You are cleaning and reshaping tabular data on its way into R, pandas, a
  database, or a plotting tool.
- You want one static binary you can `scp` to a locked-down box with no runtime.

**Avoid when:**
- Your problem is relational — multi-key joins, indexes, query planning — where
  SQLite or DuckDB is the right tool.
- You need to transform deeply nested JSON; jq's tree model fits that better.
- You want random-access dataframe semantics (pivot, window, column algebra at
  scale); Polars or pandas are built for it.
- A `cut`/`awk` one-liner already solves it — Miller's parsing overhead buys you
  nothing there.

## Alternatives

- stedolan/jq — use instead when the data is deeply nested JSON rather than
  flat/tabular records.
- BurntSushi/xsv — use instead when you only touch CSV and want maximum raw
  speed from a Rust tool (see also its successor xan).
- duckdb/duckdb — use instead when you want SQL, real joins, and Parquet/columnar
  performance over large datasets.
- pola-rs/polars — use instead when you need an in-process dataframe with pivots,
  window functions, and column algebra.
- gawk (GNU awk) — use instead when line- and position-based processing is
  sufficient and you don't need format-awareness.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2015 | First public release; original C implementation[^1]. |
| 5.x | ~2017–2021 | Mature C era: RFC-4180 CSV, expanded DSL and verb set. |
| 6.0 | 2021 | Full rewrite from C to Go; zero-dependency single binary[^2]. |
| 6.x | 2022–2026 | Current line; ongoing active maintenance under `.../v6`. |

## References

[^1]: Miller README — "What is Miller?", installation and build notes. https://github.com/johnkerl/miller
[^2]: Miller documentation — features, streaming model, and the Go rewrite. https://miller.readthedocs.io/

## Tags

cli, go, csv, tsv, json, data-processing, data-cleaning, etl, streaming, tabular-data, unix-toolkit, statistics
