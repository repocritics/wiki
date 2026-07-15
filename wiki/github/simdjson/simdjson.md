# simdjson/simdjson

> A JSON parser that uses SIMD instructions and microparallel algorithms to parse at gigabytes per second, with full validation and no compromises.

[GitHub repo](https://github.com/simdjson/simdjson) ·
[Official website](https://simdjson.org) ·
[License: Apache-2.0](https://github.com/simdjson/simdjson/blob/master/LICENSE) (also dual-licensed MIT)

## Overview

simdjson is a C++ library for parsing JSON that treats parsing as a throughput problem rather than a convenience problem. It applies commonly available SIMD instructions (SSE4.2, AVX2, AVX-512, ARM NEON) and branch-light algorithms to fully validate and parse JSON at multiple gigabytes per second on a single core[^1]. The project came out of academic research by Daniel Lemire and Geoff Langdale and remains research-adjacent: its design decisions are published in peer-reviewed venues (VLDB Journal, Software: Practice and Experience)[^1][^2].

The defining tradeoff is that simdjson is fast because it is *not* a general-purpose document object model. Its primary front-end, On-Demand, parses lazily as you navigate the document and does not build a mutable tree you can edit and re-serialize[^2]. You get validation and read speed; you give up the ergonomics of `doc["a"]["b"] = 3`. A classic DOM API also exists but is not where the performance story lives. This makes simdjson excellent as a read/ingest parser and awkward as a JSON manipulation library.

It is not a niche curiosity: it is embedded in Node.js, ClickHouse, Meta's Velox, Apache Doris, StarRocks, Milvus, QuestDB, and WatermelonDB, among others[^3]. Bindings and ports exist for Python, Rust, Go, Ruby, PHP, Java, C#, Zig, and more — most wrap the C++ core rather than reimplementing it.

## Getting Started

The library ships as a single-header amalgamation (`simdjson.h` + `simdjson.cpp`), which is the intended way to vendor it. Package-manager builds (vcpkg, Conan, Homebrew, apt) and CMake `FetchContent` are also supported.

```bash
wget https://raw.githubusercontent.com/simdjson/simdjson/master/singleheader/simdjson.h \
     https://raw.githubusercontent.com/simdjson/simdjson/master/singleheader/simdjson.cpp
```

```cpp
#include "simdjson.h"
using namespace simdjson;

int main() {
    ondemand::parser parser;
    padded_string json = padded_string::load("twitter.json");
    ondemand::document doc = parser.iterate(json);
    uint64_t count = doc["search_metadata"]["count"];
    std::cout << count << " results\n";
}
```

```bash
c++ -std=c++17 -O2 -o quickstart quickstart.cpp simdjson.cpp
```

Requires a 64-bit platform and a reasonably modern compiler (GCC 7+ or Clang 6+); C++11 is the floor, but C++17 unlocks the ergonomic error-handling paths.

## Architecture / How It Works

simdjson works in two logical stages. Stage 1 scans the entire input with SIMD to find structural characters (braces, brackets, colons, commas, quotes) and to validate UTF-8, producing an index of "interesting" byte offsets. It validates UTF-8 in less than one instruction per byte using a lookup-table approach described in a dedicated paper[^4]. Stage 2 walks that structural index to interpret values. Because stage 1 is branch-light and vectorized, it avoids the per-character branching that dominates conventional recursive-descent parsers.

There are two front-end APIs:

- **On-Demand** (`ondemand::`) — the default and fastest. It is a forward-only, lazy iterator: values are materialized as you request them, numbers and strings are parsed only when accessed, and the document is not retained as a tree. This is what enables the headline throughput but imposes usage rules (see Production Notes)[^2].
- **DOM** (`dom::`) — the older eager parser that builds a read-only document tree. Slower and more allocating, but random-access and easier to reason about. Still maintained.

**Runtime CPU dispatch** is a core design feature: the binary contains multiple compiled implementations (e.g. `icelake`/AVX-512, `haswell`/AVX2, `westmere`/SSE4.2, `arm64`/NEON, plus a scalar fallback) and selects the best one for the host CPU at load time. You compile once and ship one binary that adapts across machines; no `-march=native` required[^5].

Input requires `SIMDJSON_PADDING` bytes of slack past the logical end of the buffer so the SIMD loads never read out of bounds — hence the `padded_string` type. Feeding simdjson an unpadded buffer is the most common integration mistake.

## Production Notes

- **On-Demand is forward-only and single-pass.** You may iterate an array or object once, in order. Reading fields out of order, or re-reading them, can silently reparse from the wrong position or throw. If you need random access, either restructure your access to match document order or use the DOM API. This is the single largest source of surprise for new users.
- **Lifetime coupling.** On-Demand values are views into the source buffer and the parser. The `padded_string` (or your backing buffer) and the `parser` must outlive every value, document, and string_view you extract. Returning a `string_view` from a scope that drops the buffer is a use-after-free, not a compile error.
- **The parser is not thread-safe and is reusable by design.** A `parser` allocates internal buffers sized to the largest document it has seen; reuse one parser per thread across many documents to amortize allocation. Sharing a parser across threads is a data race.
- **Padding is mandatory.** Parsing a raw `std::string`/`char*` without `SIMDJSON_PADDING` trailing bytes is undefined behavior. Use `padded_string`, `padded_string_view`, or reserve the slack yourself.
- **Error handling has two modes.** The exception-free path returns `simdjson_result<T>` and error codes you must check; the ergonomic path uses exceptions. Mixing them carelessly (ignoring an error code, then dereferencing) is a footgun. Pick one convention per codebase.
- **Numbers are parsed exactly.** simdjson does correct round-trip float parsing (bundling a Dragonbox implementation for serialization)[^6], so it will reject inputs some lax parsers accept. This is a feature for correctness and occasionally a migration surprise.
- **Bulk/NDJSON.** For newline-delimited JSON there is a dedicated `parse_many` path with multithreaded parsing that sustains multi-GB/s; do not loop `iterate` per line if you care about throughput.

## When to Use / When Not

**Use when:**
- You ingest large volumes of JSON and parsing is a measured bottleneck (log pipelines, analytics engines, databases, RPC decoding).
- You need strict, standards-conformant JSON and UTF-8 validation with exact number handling.
- You read data and rarely mutate it, and you can respect forward-only access.

**Avoid when:**
- You need a mutable document you build up and serialize back out — On-Demand is the wrong shape and the DOM API is not its strong suit; a mutable-DOM parser fits better.
- Your JSON payloads are tiny and infrequent — parser setup, padding, and lifetime rules cost more than they save; `nlohmann/json` is simpler.
- You are not in C++ and don't want to maintain a native binding across platforms — a native parser in your own language may be less operational overhead than the SIMD win is worth.

## Alternatives

- Tencent/rapidjson — long-standing fast C++ DOM/SAX parser with in-place mutation; use when you need a mutable document tree and broad compiler support.
- nlohmann/json — the ergonomic standard for C++ JSON; use when developer experience and STL-native feel matter more than throughput.
- ibireme/yyjson — very fast C library with both a read-only and a mutable DOM; use when you want most of the speed with a mutable tree, or need a C (not C++) dependency.
- boostorg/json — Boost.JSON, a well-integrated DOM parser; use inside Boost-heavy codebases that value consistency over peak speed.
- minio/simdjson-go — a Go port of the same algorithm; use when you want simdjson-style parsing without a cgo dependency on the C++ core.

## History

| Version | Date | Notes |
|---------|------|-------|
| paper | 2019 | "Parsing Gigabytes of JSON per Second" (VLDB Journal); DOM parser with AVX2/SSE4.2 runtime dispatch[^1]. |
| 0.3 | 2020 | Major internal rewrite; broader CPU targets and API changes. |
| 0.6 | 2020 | On-Demand front-end introduced as a faster, lazy alternative to DOM. |
| 1.0 | 2021 | API stabilization; single-header amalgamation as the recommended distribution. |
| 2.0 | 2022 | On-Demand established as the primary API; refinements and AVX-512 support. |
| 3.0 | 2023 | Current major line; builder/serialization work, ongoing platform support (RISC-V, LoongArch). |
| — | 2024 | On-Demand design published in Software: Practice and Experience[^2]. |

## References

[^1]: Geoff Langdale, Daniel Lemire, "Parsing Gigabytes of JSON per Second," VLDB Journal 28(6), 2019. https://arxiv.org/abs/1902.08318
[^2]: John Keiser, Daniel Lemire, "On-Demand JSON: A Better Way to Parse Documents?" Software: Practice and Experience 54(6), 2024. https://arxiv.org/abs/2312.17149
[^3]: simdjson README, "Real-world usage." https://github.com/simdjson/simdjson#real-world-usage
[^4]: John Keiser, Daniel Lemire, "Validating UTF-8 In Less Than One Instruction Per Byte," Software: Practice & Experience 51(5), 2021. https://arxiv.org/abs/2010.03090
[^5]: simdjson docs, "Implementation Selection." https://github.com/simdjson/simdjson/blob/master/doc/implementation-selection.md
[^6]: Junekey Jeon, Dragonbox (bundled for float serialization). https://github.com/jk-jeon/dragonbox

## Tags

cpp, json, json-parser, simd, avx2, avx512, arm-neon, high-performance, parsing, utf-8-validation, header-only, database-infrastructure
