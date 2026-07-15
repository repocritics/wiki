# jbeder/yaml-cpp

> A YAML 1.2 parser and emitter for C++ with a value-semantics `Node` DOM — the long-standing default, warts and all.

[GitHub repo](https://github.com/jbeder/yaml-cpp) ·
[Tutorial (GitHub wiki)](https://github.com/jbeder/yaml-cpp/wiki/Tutorial) ·
[License: MIT](https://github.com/jbeder/yaml-cpp/blob/master/LICENSE)

## Overview

yaml-cpp is a C++ library that parses YAML into an in-memory node graph and emits YAML back out, targeting the YAML 1.2 specification[^spec]. It began on Google Code and migrated to GitHub in 2015[^created]. For over a decade it has been the reflexive choice when a C++ project needs YAML config — it is packaged by every major distro, vcpkg, conan, and Homebrew, and is what most tutorials assume.

The library has two eras. The **old API** (0.3.x) exposed a parser/event model where you streamed a document into typed nodes with `operator>>`. The **new API** (0.5.0, 2012 onward) replaced it with `YAML::Node`, a JSON-library-style handle you index with `[]` and convert with `.as<T>()`. The README states the old API stops receiving bugfixes in 2026[^readme]; new code should use the `Node` API exclusively.

The defining tension is maturity versus momentum. yaml-cpp is stable, ubiquitous, and battle-tested, but it is effectively a single-maintainer project with multi-year gaps between releases, it drops comments and most formatting on round-trip, and its `Node` copy semantics surprise nearly everyone the first time. It is a fine reader of trusted config and a poor tool for rewriting human-edited YAML in place.

## Getting Started

Install via a package manager, or vendor it with CMake `FetchContent`:

```sh
# Debian/Ubuntu: apt install libyaml-cpp-dev
# macOS:        brew install yaml-cpp
# vcpkg:        vcpkg install yaml-cpp
```

```cmake
find_package(yaml-cpp REQUIRED)
target_link_libraries(app PRIVATE yaml-cpp::yaml-cpp)
```

```cpp
#include <yaml-cpp/yaml.h>
#include <iostream>

int main() {
    YAML::Node config = YAML::LoadFile("config.yaml");

    auto name = config["name"].as<std::string>();
    int  port = config["server"]["port"].as<int>(8080);  // fallback if absent

    config["server"]["port"] = 9090;   // mutates the in-memory graph

    YAML::Emitter out;
    out << config;                     // comments/formatting are NOT preserved
    std::cout << out.c_str() << "\n";
}
```

## Architecture / How It Works

A parse (`YAML::Load`, `LoadFile`, `LoadAll`) walks the input and builds a graph of nodes owned by a reference-counted memory holder. `YAML::Node` is not the data — it is a **handle** into that shared graph. Every node has one of five types: `Null`, `Scalar`, `Sequence`, `Map`, or `Undefined`.

Two consequences follow from the handle design, and both are frequent bug sources:

- **Copies are shallow.** `YAML::Node b = a;` does not deep-copy; `b` and `a` alias the same underlying node. Assigning through `b` mutates what `a` sees. `YAML::Clone(node)` is the explicit deep copy.
- **`operator[]` inserts on access.** On a non-const node, indexing a missing key (`node["missing"]`) can materialize an `Undefined` child and, in mutation paths, grow the map. Reading is not always as passive as it looks; guard with `IsDefined()` / `IsNull()`.

Scalar conversion goes through a customization point: `YAML::convert<T>`, a template with `static Node encode(const T&)` and `static bool decode(const Node&, T&)`. Built-in specializations cover the arithmetic types, `std::string`, `std::vector`, `std::map`, etc.; you specialize it for your own structs. `.as<T>()` throws `YAML::TypedBadConversion` on failure, while the `.as<T>(fallback)` overload returns a default instead.

Anchors (`&`) and aliases (`*`) are resolved during parse, so the resulting graph can share sub-nodes. Emission runs through `YAML::Emitter`, a stream that accepts either a `Node` or a sequence of manipulators (`YAML::BeginMap`, `YAML::Key`, `YAML::Value`, `YAML::Flow`, ...). The emitter reconstructs YAML from the node graph — it does not replay the original byte stream, which is why comments and layout are lost.

## Production Notes

**Release cadence is slow.** Minor versions have landed roughly every 2–5 years: 0.6.0 (2018), 0.7.0 (2021), 0.8.0 (2023), 0.9.0 (2026). Bug fixes can sit unreleased on `master` for a long time; distros frequently ship a version one or two minors behind. Pin an exact tag if reproducibility matters.

**No comment or format preservation.** Load-modify-emit produces semantically equivalent YAML with comments stripped and formatting normalized. Do not use yaml-cpp to edit a human's config file in place — you will silently delete their comments. There is no comment-preserving C++ YAML library with yaml-cpp's ubiquity; if round-trip fidelity is the goal, reconsider the format (TOML) or the approach.

**Exceptions are mandatory.** Parse and conversion errors are reported by throwing (`YAML::ParserException` carries a `Mark` with line/column; `YAML::BadConversion` for `.as<T>()`). There is no error-code API. A `-fno-exceptions` codebase cannot use this library safely.

**Not thread-safe on a shared document.** Because nodes are handles into one shared graph and `operator[]` can mutate, concurrent access to nodes from the same `Load` — even seemingly read-only access — is unsafe. Treat a parsed document as single-threaded, or `Clone` per thread.

**Untrusted input needs guarding.** yaml-cpp has had security advisories over its history (heap issues, stack exhaustion / denial-of-service on deeply nested or alias-heavy input). It does not impose depth or alias-expansion limits for you. Do not parse adversarial YAML without bounding input size and nesting first, and keep the version current.

**ABI is not stable across versions.** There is no ABI guarantee between minor releases; a static library is built by default (`-DYAML_BUILD_SHARED_LIBS=ON` for shared). Rebuild dependents when you bump the version rather than swapping a `.so`.

## When to Use / When Not

**Use when:**
- You need to read trusted YAML config into typed C++ with minimal ceremony.
- You want the most widely packaged, best-documented C++ YAML option.
- Your build already uses CMake and can consume `yaml-cpp::yaml-cpp` directly.
- You need custom type conversion via a clean `convert<T>` extension point.

**Avoid when:**
- You must preserve comments/formatting when rewriting user-edited YAML.
- You compile with exceptions disabled.
- You are parsing untrusted input and cannot add your own resource limits.
- Parse throughput or allocation pressure is critical (large documents, hot loops, embedded).

## Alternatives

- biojppm/rapidyaml — much faster and lower-allocation with in-situ parsing; use when parse throughput or memory footprint dominates and you can accept a lower-level API.
- yaml/libyaml — the C reference parser (YAML 1.1, event-based, no DOM); use when you need a C dependency or a backend to build your own model on.
- fktn-k/fkYAML — header-only modern C++ with a JSON-like node API; use when you want a single-header drop-in with no build integration.
- nlohmann/json — use instead when you only actually need JSON and want first-class ergonomics rather than YAML's surface area.
- marzer/tomlplusplus — use when the file is config a human edits and you would rather adopt TOML, which round-trips and comments more predictably.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.3.0 | 2012-01-21 | Last release of the old parser/event API[^readme]. |
| 0.5.0 | 2012-11-09 | New `YAML::Node` value-semantics API introduced. |
| 0.6.0 | 2018-01-28 | Requires C++11; CMake modernization. |
| 0.6.3 | 2019-09-25 | Long-lived maintenance baseline for many distros. |
| 0.7.0 | 2021-07-10 | Build/packaging fixes, `yaml-cpp::yaml-cpp` target conventions. |
| 0.8.0 | 2023-08-10 | Continued fixes; CMake `FetchContent` guidance. |
| 0.9.0 | 2026-02-04 | Current release[^readme]. |

## References

[^spec]: YAML 1.2 specification. https://yaml.org/spec/1.2.2/
[^created]: GitHub repository metadata, `created_at` 2015-03-30 (project predates GitHub; migrated from Google Code). https://github.com/jbeder/yaml-cpp
[^readme]: yaml-cpp README — build, integration, and "old API stops receiving bugfixes in 2026" note. https://github.com/jbeder/yaml-cpp/blob/master/README.md

## Tags

yaml, parser, serialization, cpp, config, emitter, dom, cmake, yaml-1.2, library
