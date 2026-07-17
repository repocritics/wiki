# p-ranav/csv2

> Header-only, memory-mapped CSV reader and writer for C++11 that trades a rich feature set for raw scan speed.

[GitHub repo](https://github.com/p-ranav/csv2) ·
[License: MIT](https://github.com/p-ranav/csv2/blob/master/LICENSE)

## Overview

csv2 is a single-header C++11 library for reading and writing delimiter-separated files, written by Pranav Srinivas Kumar (p-ranav), the author of a family of modern-C++ single-header utilities including argparse and indicators[^1]. Its reader is built around a memory-mapped file and lazy, iterator-based traversal: you `mmap` a file and walk rows and cells without the library allocating a copy of the data. The design goal is throughput on large files, and the README reports whole-file scans of multi-gigabyte datasets in single-digit seconds on commodity hardware[^2].

The defining tension is scope. csv2 parses structure — rows, cells, quoting, trimming — but does almost nothing above that line. There is no typed column access, no header-name lookup, no schema, no streaming from arbitrary `istream`s, and no built-in type conversion beyond handing you the raw or unescaped bytes of a cell. You get a fast cursor over the grid and are expected to do everything else yourself. For code that just needs to iterate cells as strings this is a good fit; for anything resembling a dataframe it is a foundation, not a solution.

Maintenance status is the other thing a reader should weigh. The last commit to the default branch was in December 2023[^3], and the open-issue count is modest but non-trivial. Treat csv2 as a small, stable, effectively finished library rather than an actively developed one — fine for a dependency you vendor and freeze, less ideal if you expect upstream fixes.

## Getting Started

csv2 is header-only. Vendor the `include/` directory (or the generated `single_include/csv2/csv2.hpp`) and point your compiler at it — no build step, no link step. It is also packaged for vcpkg and Conan.

```cpp
#include <csv2/reader.hpp>
#include <string>

int main() {
  csv2::Reader<csv2::delimiter<','>,
               csv2::quote_character<'"'>,
               csv2::first_row_is_header<true>,
               csv2::trim_policy::trim_whitespace> csv;

  if (csv.mmap("foo.csv")) {
    const auto header = csv.header();
    for (const auto row : csv) {
      for (const auto cell : row) {
        std::string value;
        cell.read_value(value);   // unescaped cell contents
      }
    }
  }
}
```

Compile with `-std=c++11` (or later) and, for realistic numbers, `-O3`. The reader's behavior is fixed at compile time through template parameters — delimiter, quote character, header handling, and trim policy are all types, not runtime arguments.

## Architecture / How It Works

The reader `mmap`s the target file into the process address space and treats the mapping as a flat `char` buffer. Iteration is lazy: `Reader::begin()` returns a `RowIterator` that scans forward for the next row terminator, and each `Row` exposes a `CellIterator` that scans for the next delimiter, honoring the configured quote character. Nothing is parsed ahead of the cursor, and cells are returned as offsets into the mapped buffer rather than as copied strings. This is what makes large-file scans fast and low-allocation: the OS pages the file in on demand and the library never materializes the whole grid.

Cells expose two accessors. `read_raw_value` copies the bytes exactly as they appear in the file; `read_value` additionally unescapes doubled quote characters (the CSV convention where `""` inside a quoted field means a literal `"`). Both write into a caller-supplied container, so the caller owns allocation.

Configuration lives entirely in the type: `trim_policy`, `first_row_is_header`, `delimiter<';'>`, and `quote_character` are template parameters, so each distinct configuration is a distinct instantiated type with no runtime dispatch in the hot loop. The cost is that you cannot decide the delimiter at runtime from the same `Reader` type — a real limitation if your program must sniff formats.

The writer is a much thinner facility: a `Writer<delimiter<','>>` wraps an `std::ofstream` and offers `write_row` and `write_rows` over containers of strings. It does the delimiter joining and newline handling; quoting and escaping of embedded delimiters or quotes are largely the caller's responsibility.

## Production Notes

**mmap has hard constraints.** The input must be a real, seekable file — you cannot map a socket, pipe, `stdin`, or in-memory `istream`. There is a `parse(std::string)` path for data you already hold, but that keeps the whole payload in a `std::string` and gives up the mapping benefit. Mapping also ties process stability to the file: truncation by another process while mapped can fault your program.

**Correctness is not the same as full RFC 4180 conformance.** csv2 handles the common quoting and escaping cases, but the spec has edge cases (embedded newlines inside quoted fields, mixed line endings, BOM handling, ragged rows) where behavior should be verified against your actual data before trusting it in a pipeline. The single-threaded design also means you scale by sharding files across threads yourself; the library will not parallelize a single file for you.

**No typed access.** Every cell comes out as bytes. Numeric parsing, date parsing, and header-name-to-column mapping are all on you. For wide analytical files this adds meaningful glue code compared with a dataframe-oriented library.

**Compile-time configuration is a double-edged sword.** Fixing delimiter and quoting in the type gives a tight inner loop but forces template instantiation for every format variant and rules out runtime format detection with a single reader type. Heavy use across many configurations can inflate compile times and binary size.

**Dormancy.** With no upstream activity since late 2023[^3], plan to own any fixes yourself. Vendoring the single header and pinning a known-good revision is the pragmatic posture; do not assume reported bugs will be addressed upstream.

## When to Use / When Not

**Use when:**
- You need to scan large, well-formed CSV files fast with minimal allocation.
- Your input is always an on-disk file you can `mmap`.
- You want a zero-build, header-only dependency you can vendor and freeze.
- You are comfortable doing type conversion and column mapping yourself.

**Avoid when:**
- You need to parse from streams, sockets, stdin, or compressed input.
- You want typed columns, header-name access, or a dataframe-like API.
- You must choose the delimiter or quoting at runtime from one code path.
- You need an actively maintained parser with responsive upstream fixes.
- You require strict, audited RFC 4180 conformance out of the box.

## Alternatives

- ben-strasser/fast-cpp-csv-parser — header-only with compile-time-typed column reading; use it when you want strongly-typed columns rather than raw cells.
- vincentlaucsb/csv-parser — richer, RFC-4180-focused C++ parser with type inference and stream support; use it when correctness and convenience beat raw mmap speed.
- d99kris/rapidcsv — single-header, container-oriented (load into vectors, index by row or column name); use it for small-to-medium files and ergonomic access.
- AriaFallah/csv-parser — small streaming C++ parser; use when you must consume from an `istream` rather than a mapped file.
- apache/arrow CSV reader — use when CSV is one input into a columnar analytics pipeline and you want multithreaded, typed ingestion.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2020-04-17 | Repository created[^4]. |
| benchmark refresh | 2022-09-23 | README benchmark results re-measured on 11th-gen Intel hardware[^2]. |
| last commit | 2023-12-23 | Most recent activity on the default branch[^3]. |

csv2 does not publish tagged semantic-version releases; the table tracks repository milestones rather than release numbers.

## References

[^1]: p-ranav — GitHub profile and single-header C++ libraries. https://github.com/p-ranav
[^2]: csv2 README — "Performance Benchmark," results dated 23 SEP 2022. https://github.com/p-ranav/csv2#performance-benchmark
[^3]: csv2 repository — commit history on `master`; most recent push 2023-12-23 (via GitHub API `pushed_at`). https://github.com/p-ranav/csv2/commits/master
[^4]: csv2 repository — created 2020-04-17 (GitHub API `created_at`). https://github.com/p-ranav/csv2

## Tags

cpp, cpp11, csv-parser, csv-writer, header-only, single-header, memory-mapped, mmap, lazy-evaluation, parsing, data-ingestion
