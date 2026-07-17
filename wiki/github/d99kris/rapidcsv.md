# d99kris/rapidcsv

> A single-header C++11 CSV library that loads the whole file into memory as a mutable, spreadsheet-like document.

[GitHub repo](https://github.com/d99kris/rapidcsv) ·
[License: BSD-3-Clause](https://github.com/d99kris/rapidcsv/blob/master/LICENSE)

## Overview

Rapidcsv is a header-only C++ CSV parser by Kristofer Berggren, first published
in 2017[^1]. Despite the name, it is less a streaming "parser" and more an
in-memory document model: you construct a `rapidcsv::Document` from a file,
stream, or string, and it holds every cell as a `std::string` in a row-major
grid. You then read and write cells, rows, and columns by label or index, with
type conversion applied on access. The whole surface is one file,
`src/rapidcsv.h`, that you copy into your include path.

The library's defining tension is convenience versus throughput. Storing every
cell as a string and converting on each `Get` call makes the API pleasant —
`doc.GetColumn<float>("Close")` reads like a spreadsheet formula — but means
the file is fully resident in memory (often larger than on disk once split into
`std::string` objects) and that numeric columns are re-parsed each time they
are requested. Rapidcsv is aimed at small-to-medium CSVs where developer time
matters more than parse time: config tables, financial time series, test
fixtures, quick data munging. It was featured in Deitel's *C++20 for
Programmers*[^2], which reflects its role as a teaching-grade, easy-to-adopt
library rather than a high-throughput ingestion engine.

## Getting Started

Rapidcsv is header-only. The canonical install is to copy the single header:

```bash
# vendored copy
curl -O https://raw.githubusercontent.com/d99kris/rapidcsv/master/src/rapidcsv.h
```

It is also packaged for [vcpkg](https://vcpkg.io) (`rapidcsv`) and
[Conan](https://conan.io/center/rapidcsv), and integrates via CMake
`add_subdirectory`, `FetchContent`, or `find_package`[^3].

```cpp
#include <iostream>
#include <vector>
#include "rapidcsv.h"

int main()
{
  rapidcsv::Document doc("data.csv");                 // first row = column headers
  std::vector<float> close = doc.GetColumn<float>("Close");
  std::cout << "Read " << close.size() << " values.\n";

  // Row/cell access needs a row-header column: LabelParams(colHdrIdx, rowHdrIdx)
  rapidcsv::Document keyed("data.csv", rapidcsv::LabelParams(0, 0));
  long long vol = keyed.GetCell<long long>("Volume", "2017-02-22");
  std::cout << "Volume " << vol << "\n";
}
```

## Architecture / How It Works

The entire library is one class graph in `rapidcsv.h`. `Document` owns
`mData`, a `std::vector<std::vector<std::string>>` holding every cell verbatim.
Parsing happens once at construction: the reader walks the input byte by byte,
honoring quoting, embedded separators, and (optionally) quoted line breaks,
then materializes the grid. There is no lazy or streaming mode — construction
cost and memory are proportional to the whole file.

Typed access is where the design shows. `GetCell<T>`, `GetColumn<T>`, and
`GetRow<T>` template on the target type and route through `Converter<T>`, which
calls the standard conversion routines (`std::stof`, `std::stoll`, etc.) on the
stored string each call. `char` is a documented special case — the cell's first
byte is returned rather than a parsed number. You can override conversion
globally by specializing `Converter<T>::ToVal`/`ToStr`, or per call by passing
a conversion function or lambda to the `Get`/`Set` methods. This gives clean
extension points for custom types and fixed-point money, at the cost of doing
the parse work again on every read.

Behavior is tuned through a set of params structs passed to the constructor:
`LabelParams` (which row/column act as headers; `-1` means "none"),
`SeparatorParams` (delimiter, trimming, CR handling, quoted line breaks,
auto-dequoting), `ConverterParams` (invalid-number handling, locale mode), and
`LineReaderParams` (skip comment/empty lines). The document is fully mutable —
`SetCell`, `InsertRow`, `RemoveColumn`, and friends modify `mData` in place —
and `Save()` re-serializes it, so rapidcsv doubles as a small CSV writer and
editor, not only a reader.

## Production Notes

- **Memory, not streaming.** The full file lives in RAM as split
  `std::string`s. A file that is a few hundred MB on disk can balloon past that
  in memory once every field is a separate string object. For multi-GB inputs,
  rapidcsv is the wrong tool — reach for a streaming parser.
- **Re-parsing on every access.** `GetColumn<float>` converts the whole column
  each time it is called; there is no cached typed view. In hot loops, fetch
  once into a `std::vector<T>` and reuse it.
- **Locale-dependent float parsing by default.** Numeric conversion uses the
  active C locale, so a machine set to a comma-decimal locale can misparse
  dot-decimal data (and vice versa). Set `mNumericLocale` in `ConverterParams`
  for locale-independent parsing[^4].
- **Exceptions on bad numeric cells.** Reading a non-numeric or empty cell as a
  number throws by default (it propagates `std::stof`/`std::stoll` exceptions).
  Pass `ConverterParams(true)` to substitute a default (signaling NaN for
  floats, 0 for integers) instead — but that silently masks dirty data.
- **UTF-16 is opt-in and codecvt-dependent.** UTF-8 is the preferred encoding;
  UTF-16 LE/BE read and write require defining `HAS_CODECVT`, which relies on
  `<codecvt>` — deprecated since C++17 and absent on some toolchains. The test
  build auto-detects it.
- **Not RFC-validating.** Quoting, embedded commas, and quoted line breaks are
  handled, but malformed rows with inconsistent column counts are accepted
  as-is. Sharing a mutable `Document` across threads is also unsupported (a
  read-only `const` document is fine).

## When to Use / When Not

**Use when:**
- CSVs are small to medium and fit comfortably in memory.
- You want random access by column/row label and an easy, readable API.
- You need to read *and* write/edit CSV, or generate CSV from a grid.
- You want a zero-dependency, single-header drop-in with C++11 support.

**Avoid when:**
- Files are very large or unbounded — you need a streaming/incremental parser.
- Throughput is critical and the schema is fixed at compile time.
- You need columnar analytics, typed columns, or out-of-core processing.
- You require strict RFC 4180 validation of malformed input.

## Alternatives

- ben-strasser/fast-cpp-csv-parser — compile-time column typing and a
  streaming reader; use it when parse speed matters and the schema is known
  ahead of time.
- vincentlaucsb/csv-parser — streaming, statistics, and broader CSV edge-case
  handling; use it for larger files or messier real-world CSV.
- p-ranav/csv2 — lazy, low-allocation reader tuned for speed; use it for fast
  read-only passes over big files.
- apache/arrow — columnar, typed, out-of-core CSV ingestion; use it when CSV is
  a feed into analytics rather than the end format.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2017-05 | Header-only C++11 CSV Document model published[^1]. |
| — | 2020+ | vcpkg/Conan packaging; featured in Deitel *C++20 for Programmers*[^2]. |
| — | 2026-06 | Still maintained; latest commit on `master` mid-2026[^5]. |

## References

[^1]: rapidcsv repository — created 2017-05-19. https://github.com/d99kris/rapidcsv
[^2]: Deitel, *C++20 for Programmers*. https://deitel.com/c-plus-plus-20-for-programmers/
[^3]: rapidcsv README, Installation / CMake sections. https://github.com/d99kris/rapidcsv#installation
[^4]: rapidcsv README, "Locale Independent Parsing"; see tests/test087.cpp. https://github.com/d99kris/rapidcsv
[^5]: GitHub API repo metadata, `pushed_at` 2026-06-14 (fetched 2026-07). https://api.github.com/repos/d99kris/rapidcsv

## Tags

cpp, cpp11, csv, csv-parser, header-only, single-header, data-parsing, library, bsd-licensed, in-memory
