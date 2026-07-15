# nlohmann/json

> A single-header JSON library for C++ that trades raw speed for a syntax that reads like a scripting language.

[GitHub repo](https://github.com/nlohmann/json) ·
[Official website](https://json.nlohmann.me) ·
[License: MIT](https://github.com/nlohmann/json/blob/develop/LICENSE.MIT)

## Overview

nlohmann/json ("JSON for Modern C++") is a header-only C++11 library by Niels Lohmann, first released in 2016[^1]. Its defining bet is ergonomics over performance: JSON values behave like a first-class dynamic type, so `j["user"]["name"] = "Tom"` and `json j = {{"pi", 3.141}, {"list", {1, 0, 2}}}` compile to a real `std::map`/`std::vector`-backed structure. For a language without a native JSON type, this is the closest C++ gets to Python's `dict`.

The library is explicit that speed and memory efficiency were *not* design goals[^2]. Each value carries a tagged-union overhead, the default object type is an alphabetically-ordered `std::map`, and parsing is materially slower than the SAX/DOM libraries built for throughput. What it buys instead is trivial integration (one `#include`, no build system, no dependencies) and unusually thorough testing — 100% line coverage plus continuous fuzzing through Google OSS-Fuzz[^3].

That tradeoff defines who it is for. It is the default choice for application code, config parsing, tooling, and tests where developer time dominates and JSON is not the bottleneck. It is the wrong choice for parsing gigabytes per second. The project is broadly adopted (packaged in effectively every C++ package manager) and remains actively maintained, though release cadence has slowed to roughly annual on the 3.x line.

## Getting Started

The whole library is one header. Vendoring `single_include/nlohmann/json.hpp` and adding it to your include path is a complete install. With CMake:

```cmake
include(FetchContent)
FetchContent_Declare(json URL https://github.com/nlohmann/json/releases/download/v3.11.3/json.tar.xz)
FetchContent_MakeAvailable(json)
target_link_libraries(my_app PRIVATE nlohmann_json::nlohmann_json)
```

```cpp
#include <nlohmann/json.hpp>
using json = nlohmann::json;

int main() {
    // build by assignment — types are inferred
    json j;
    j["name"] = "Tom";
    j["scores"] = {90, 85, 100};
    j["active"] = true;

    // parse from a string
    json parsed = json::parse(R"({"pi": 3.141, "nested": {"x": 42}})");
    int x = parsed["nested"]["x"].get<int>();

    // serialize (2-space pretty print)
    std::cout << j.dump(2) << '\n';
}
```

Round-tripping to your own structs uses ADL hooks or a macro:

```cpp
struct User { std::string name; int age; };
NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE(User, name, age)

json j = User{"Tom", 39};
User u = j.get<User>();
```

## Architecture / How It Works

The public `json` type is a typedef of the `basic_json` class template, parameterized on the container types used for objects, arrays, strings, and the three number representations. Internally every value is a tagged union (a `value_t` enum tag plus a union of pointers/scalars), which is why the memory overhead per node is real but bounded.

- **Number model.** JSON numbers are stored as one of `int64_t`, `uint64_t`, or `double`. There is no arbitrary-precision path, so integers beyond 64 bits and high-precision decimals lose fidelity on parse. `dump()` uses Grisu2 for shortest round-trippable float output.
- **Object ordering.** `nlohmann::json` uses `std::map`, so object keys serialize in *sorted* order, not insertion order. If you need to preserve source order, use the separate `nlohmann::ordered_json` typedef[^4], which swaps in an insertion-ordered container.
- **Serialization is via ADL.** Custom types plug in by defining `to_json`/`from_json` free functions in the type's namespace, or with the `NLOHMANN_DEFINE_TYPE_*` macros. There is no reflection — every field is listed explicitly.
- **Parsing.** A DOM parser builds the full tree; a separate SAX interface (`sax_parse`) exists for streaming without materializing the whole document. Binary formats (CBOR, MessagePack, BSON, UBJSON, BJData) share the same value model through `to_*`/`from_*` functions.
- **JSON standards.** JSON Pointer (RFC 6901), JSON Patch (RFC 6902), and JSON Merge Patch (RFC 7386) are implemented in-tree, including `diff()` to compute a patch between two documents.

Compile-time cost is the structural consequence of the single-header design: `json.hpp` is ~25k lines of heavily-templated code, and every translation unit that includes it pays to instantiate it.

## Production Notes

- **Compile times are the headline complaint.** Including the full header in many translation units bloats build time noticeably. Mitigations: include `json_fwd.hpp` where only a forward declaration is needed, put JSON-touching code behind a `.cpp` boundary, and lean on precompiled headers. There is no meaningful runtime fix — it is a template-instantiation cost.
- **`operator[]` inserts silently.** Like `std::map`, non-const `operator[]` default-constructs a null value when the key is missing, mutating the object. Reading an optional field with `j["maybe"]` on a mutable `json` can silently grow it. Use `at()` (throws on missing), `value("key", default)`, or `contains()` for read paths.
- **Speed.** Independent benchmarks consistently place it well behind RapidJSON and an order of magnitude behind simdjson on parse throughput[^5]. Fine for config and RPC payloads; a poor fit for high-volume ingestion. Profile before assuming it is the bottleneck — often it is not.
- **Exceptions.** Errors are reported by throwing typed exceptions (`parse_error`, `type_error`, `out_of_range`, etc.). If you build with `JSON_NOEXCEPTION` (or `-fno-exceptions`), those throw sites become `abort()` — there is no error-code API, so you must validate with `accept()` or the SAX callbacks first.
- **Implicit conversions can surprise.** By default a `json` converts implicitly to many types, which occasionally binds the wrong overload. `JSON_USE_IMPLICIT_CONVERSIONS=0` disables this and forces explicit `.get<T>()`, which many teams prefer.
- **UTF-8 only.** Input must be valid UTF-8; `dump()` throws on invalid encoding unless you pass an error handler (`error_handler_t::replace`/`ignore`).

## When to Use / When Not

**Use when:**
- You want JSON to feel like a native dynamic type and value developer ergonomics over microseconds.
- JSON handling is config, tooling, tests, or moderate-volume RPC — not the hot path.
- You need one-file, zero-dependency integration that drops into any build.
- You need JSON Pointer / Patch / Merge Patch or binary formats (CBOR, MessagePack) behind one API.

**Avoid when:**
- Parse/serialize throughput is a bottleneck (streaming ingestion, gigabytes/sec) — reach for simdjson or RapidJSON.
- Compile time or binary size is tightly constrained across a large codebase.
- You need arbitrary-precision numbers or exact big-integer preservation.
- You build with exceptions disabled and want a real error-code API rather than `abort()`.

## Alternatives

- Tencent/rapidjson — mature, much faster DOM+SAX parser; more manual, less ergonomic API. Use when parse speed matters and you can accept a lower-level interface.
- simdjson/simdjson — SIMD-accelerated, the fastest parser available; read-oriented. Use for high-volume parsing of trusted input where you mostly read, not mutate.
- stephenberry/glaze — header-only, reflection-based, very fast serialization for known struct schemas (newer C++). Use when your data maps to fixed C++ types and you want speed without boilerplate.
- boostorg/json — Boost's JSON library with a similar DOM feel and better performance; use when you already depend on Boost.
- open-source-parsers/jsoncpp — older, simpler, widely packaged. Use when you need a conservative, long-established dependency and the modern ergonomics do not matter.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2016-01 | First stable release; single-header C++11 design[^1]. |
| 2.0.0 | 2016-07 | Semantic-versioning reset, API cleanup. |
| 3.0.0 | 2017-12 | Typed exception hierarchy, SAX parser interface, CBOR/MessagePack. |
| 3.9.0 | 2020-07 | `ordered_json` (insertion-ordered objects)[^4]. |
| 3.11.0 | 2022-08 | Reduced compile times, macro and diagnostics improvements. |
| 3.11.3 | 2023-11 | Widely-packaged patch release of the 3.11 line. |
| 3.12.0 | 2025-04 | C++20 module build support, further binary-format and tooling work. |

## References

[^1]: nlohmann/json releases. https://github.com/nlohmann/json/releases
[^2]: README "Design goals" — explicitly lists memory efficiency and speed as non-goals. https://github.com/nlohmann/json#design-goals
[^3]: Google OSS-Fuzz continuous fuzzing of the JSON parsers. https://github.com/google/oss-fuzz/tree/master/projects/json
[^4]: `nlohmann::ordered_json` documentation. https://json.nlohmann.me/api/ordered_json/
[^5]: Milo Yip, nativejson-benchmark — parsing/serialization comparison across C++ JSON libraries. https://github.com/miloyip/nativejson-benchmark

## Tags

cpp, json, header-only, serialization, parser, cbor, messagepack, json-pointer, json-patch, c-plus-plus-11, data-format
