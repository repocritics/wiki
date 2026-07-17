# ben-strasser/fast-cpp-csv-parser

> A header-only C++11 library that reads CSV files fast by overlapping disk I/O with parsing and compiling only the features you use into the template.

[GitHub repo](https://github.com/ben-strasser/fast-cpp-csv-parser) ·
No official website ·
[License: BSD-3-Clause](https://github.com/ben-strasser/fast-cpp-csv-parser/blob/master/LICENSE)

## Overview

fast-cpp-csv-parser is a single-file, header-only library (`csv.h`) for reading
delimited text in C++. It targets one job — pulling a fixed, compile-time-known
set of columns out of possibly-huge CSV/TSV files at close to disk speed — and
deliberately does not try to be a general-purpose CSV toolkit. There is no
writer, no in-memory table, no dynamic column model. You declare how many
columns you want as a template parameter and read rows into typed variables.

The design bet is that most performance-sensitive CSV work is a streaming
`select col1, col2, col3 from file.csv` and that the parser should look like
exactly that. Two ideas make it quick: a background thread reads the next file
chunk while the main thread parses the current one, and parsing behavior (quote
escaping, trimming, overflow handling, comment skipping) is chosen through
template policies so unused features cost nothing at runtime.[^1] The tension is
that this compile-time rigidity is also its ceiling: the things it refuses to do
(quoted fields spanning newlines, variable column counts, writing) are refused
by design, not oversight.

The project has been stable and lightly maintained for years rather than
actively developed. With roughly 2,400 stars and 440 forks as of 2026 it is a
well-known choice in the C++ ecosystem, but the last push was in early 2025[^2]
and the API has been essentially frozen for a long time — which for a
single-header parser is closer to "done" than "abandoned."

## Getting Started

There is nothing to build or link as a library — copy `csv.h` onto your include
path. Threads are used, so with GCC you must link pthread, and the `-lpthread`
must come last on the link line.

```bash
g++ -std=c++11 main.cpp -o prog -lpthread
```

```cpp
#include "csv.h"

int main() {
  io::CSVReader<3> in("ram.csv");
  in.read_header(io::ignore_extra_column, "vendor", "size", "speed");
  std::string vendor; int size; double speed;
  while (in.read_row(vendor, size, speed)) {
    // columns arrive in the order you named them, not the file's order
  }
}
```

`read_header` reads the first line and reorders the file's columns to match the
names you passed, so a file that reorders or adds columns still works as long as
the ones you asked for are present. To disable threading entirely (broken
`std::thread`, or you just do not want a worker thread), define
`CSV_IO_NO_THREAD` before including the header.

## Architecture / How It Works

The header exposes two classes in the `io` namespace. `LineReader` handles
efficient line-at-a-time reading — it owns the I/O, BOM skipping, and newline
normalization. `CSVReader` builds on it to split, trim, unescape, and parse
fields into typed variables.[^1]

**Overlapped I/O.** The reader pulls the file in chunks. While the main thread
parses the chunk it has, a worker thread fetches the next one, hiding read
latency behind parse time. A consequence worth knowing: the first chunk is read
synchronously, so a file small enough to fit in one chunk never spawns the async
path — which historically masked pthread-linking bugs, since small test files
"worked" while large files crashed.[^3]

**Policy templates.** `CSVReader` is parameterized by four policy classes plus
the column count:

```cpp
template<unsigned column_count,
         class trim_policy     = trim_chars<' ', '\t'>,
         class quote_policy    = no_quote_escape<','>,
         class overflow_policy = throw_on_overflow,
         class comment_policy  = no_comment>
class CSVReader;
```

Each policy is a class of static members, so the compiler inlines the chosen
behavior and generates no branches for behavior you did not select. Quote
escaping (`double_quote_escape<sep, quote>`) is opt-in precisely because it is
expensive; the default `no_quote_escape` skips it. Custom separators (tab, etc.),
custom trim character sets, overflow strategy (throw / ignore / clamp to max),
and comment skipping (`single_line_comment<'#'>`, empty-line skipping) are all
selected the same way.

**Zero-copy custom parsing.** Built-in field types cover the signed/unsigned
integers, floating point, `char`, and `std::string`. For anything else you read
a `char*`, which points directly into the internal buffer (already trimmed,
unescaped, null-terminated) and stays valid until the next `read_row`. The docs
are explicit that this is not slower than the built-in parsers — the built-ins
are convenience, not a fast path — so custom types carry no inherent
penalty.[^1] Reading into `std::string` is the slower option because it copies.

## Production Notes

- **pthread linking is the classic footgun.** On GCC the link fails or the
  program throws `std::system_error` / crashes on large files if you forget
  `-lpthread`, and it must be the last argument. Symptoms are misleading: small
  files pass, big files crash, because only the large path goes async.[^3]
- **Hard 2^24−1 character per-line limit.** Exceed it and `LineReader` throws
  `error::line_length_limit_exceeded`. Fine for normal data, a real ceiling for
  pathological single-line blobs.[^1]
- **No multi-line quoted fields.** RFC 4180 permits a newline inside a quoted
  field; this library does not support it and the maintainer has stated it is
  hard to add without breaking the current design. If your data has embedded
  newlines in quotes, this is a correctness bug for you, not a limitation you can
  configure around — pick a different parser.[^1]
- **Fixed column count only.** You read a compile-time constant number of
  columns. The file may vary its columns, but your read is a fixed projection.
  There is no way to read a genuinely variable number of columns per row.
- **UTF-8 is pass-through, not aware.** BOMs are stripped and UTF-8 bytes survive
  a `char*` read intact, but separators, quotes, and comment characters must be
  ASCII, and the library does no decoding. Multi-byte delimiters are out.
- **`has_column` inside the read loop kills performance.** It is correct there
  but the docs warn it significantly slows processing; call it once before the
  loop.
- **Error messages are user-ready.** Exceptions carry file name and line context
  and `what()` returns a formatted message; the file name is truncated internally
  (C-strings) specifically to avoid `std::bad_alloc` while reporting an error.
- **Reading only.** No CSV writing. Pair with something else if you emit CSV.

## When to Use / When Not

**Use when:**
- You know your columns at compile time and want a typed, streaming read.
- You are processing multi-GB files and want I/O overlapped with parsing for free.
- You want a zero-dependency, single-header drop-in with no build step.
- You want to pay (in code size and speed) only for the parsing features you use.

**Avoid when:**
- Your CSV has quoted fields containing newlines (unsupported by design).
- You need a truly dynamic schema, or to discover columns at runtime.
- You need to write CSV, or want an in-memory table / DataFrame abstraction.
- You need full RFC 4180 conformance or non-ASCII delimiters.

## Alternatives

- d99kris/rapidcsv — header-only C++11, far more convenient row/column/whole-file
  API; use it when ergonomics and random access matter more than streaming huge files.
- vincentlaucsb/csv-parser — C++17, RFC 4180-aware including multi-line quoted
  fields; use it when your data has embedded newlines this library rejects.
- p-ranav/csv2 — memory-mapped, multi-threaded reader focused on raw throughput;
  use it when you want maximum speed and can accept a lower-level API.
- AriaFallah/csv-parser — small header-only streaming parser; use it when you want
  a simpler codebase and do not need the policy/template machinery.
- apache/arrow — heavyweight columnar toolkit with a CSV reader; use it when CSV
  is one input to a larger analytics/columnar pipeline, not the end goal.

## History

Formal tagged releases are not published for this project; it is versioned by
git history rather than release numbers, so intermediate dates below are
approximate milestones, not official version cuts.

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2015-04-15 | Repository created; header-only C++11 reader, developed and tested against GCC 4.6.1.[^2][^1] |
| — | 2015–2024 | Incremental fixes; API and policy model kept stable, no formal releases. |
| Latest | 2025-02-02 | Most recent push; library effectively feature-frozen and lightly maintained.[^2] |

## References

[^1]: Project README — features, `LineReader`/`CSVReader` API, policy templates, limits, and FAQ. https://github.com/ben-strasser/fast-cpp-csv-parser/blob/master/README.md
[^2]: GitHub REST API, `repos/ben-strasser/fast-cpp-csv-parser` — created 2015-04-15, last push 2025-02-02, BSD-3-Clause, ~2,359 stars / 440 forks (fetched 2026-07). https://github.com/ben-strasser/fast-cpp-csv-parser
[^3]: README, Installation and FAQ — pthread linking order, `CSV_IO_NO_THREAD`, and why crashes appear only on large files (first chunk read synchronously). https://github.com/ben-strasser/fast-cpp-csv-parser/blob/master/README.md

## Tags

cpp, csv, parser, header-only, cpp11, file-io, streaming, data-ingestion, zero-dependency, template-metaprogramming
