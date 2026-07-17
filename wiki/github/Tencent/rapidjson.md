# Tencent/rapidjson

> A header-only C++ JSON parser/generator built for speed, with both SAX and DOM APIs and no STL dependency.

[GitHub repo](https://github.com/Tencent/rapidjson) ·
[Official website](http://rapidjson.org/) ·
License: MIT[^1]

## Overview

RapidJSON is a JSON library for C++ originally written by Milo Yip and later donated to Tencent's open source program[^2]. It was inspired by RapidXml and shares that library's core idea: parse in place, minimize allocations, and stay header-only so integration is a copy of one include directory. It offers two parsing models in one package — a SAX-style event `Reader`/`Writer` and a DOM-style `Document` — and is deliberately self-contained, depending on neither Boost nor the STL.

The defining tension of RapidJSON is that it is simultaneously ubiquitous and effectively frozen. It remains one of the most-depended-on C++ JSON libraries (vendored into Chromium-adjacent projects, MongoDB tooling, game engines, and countless internal codebases), yet its last tagged release, v1.1.0, shipped in August 2016[^3]. The `master` branch still receives occasional fixes — the most recent commits are from early 2025[^4] — but there has been no new release in roughly a decade, and the open issue count (well into the hundreds) reflects a maintenance backlog rather than active feature work. Teams adopt it for raw throughput and accept that they are pinning to a mature, minimally-maintained dependency.

RapidJSON's design bias is performance over ergonomics. Parsing can approach `strlen()` speed on well-formed input, with optional SSE2/SSE4.2 acceleration for whitespace skipping, and each DOM `Value` is packed into 16 bytes on 64-bit targets. The cost is an API that leaks its allocation and lifetime model into user code: values borrow from a document-owned allocator, string ownership is explicit, and misuse produces subtle memory bugs rather than compile errors.

## Getting Started

Header-only — copy `include/rapidjson` into your include path, or install via a package manager:

```bash
# vcpkg
vcpkg install rapidjson
# Or Debian/Ubuntu
apt-get install rapidjson-dev
```

```cpp
#include "rapidjson/document.h"
#include "rapidjson/writer.h"
#include "rapidjson/stringbuffer.h"
#include <iostream>

int main() {
    const char* json = R"({"project":"rapidjson","stars":10})";

    rapidjson::Document d;
    d.Parse(json);                       // DOM parse
    if (d.HasParseError()) return 1;     // errors are NOT exceptions

    d["stars"].SetInt(d["stars"].GetInt() + 1);

    rapidjson::StringBuffer buffer;
    rapidjson::Writer<rapidjson::StringBuffer> writer(buffer);
    d.Accept(writer);                    // serialize DOM via SAX events

    std::cout << buffer.GetString() << "\n";  // {"project":"rapidjson","stars":11}
}
```

## Architecture / How It Works

RapidJSON is layered so the same core drives both APIs:

1. **SAX layer** — `Reader` parses a stream and emits events (`StartObject`, `Key`, `String`, `Int`, …) to a handler you write. `Writer`/`PrettyWriter` are handlers that emit JSON text. The SAX core is only a few hundred lines and is where the throughput lives; the DOM is built on top of it.
2. **DOM layer** — `Document` is a `Value` that owns a `MemoryPoolAllocator`. Parsing populates the tree by feeding SAX events into a handler that constructs `Value` nodes from pool memory. This is why DOM parsing is fast: nodes are bump-allocated in large blocks rather than individually `new`-ed.

Key internal decisions that shape usage:

- **Allocator ownership.** A `Value` does not own its own memory; the `Document` (or an allocator you pass) does. Copying a `Value` is disabled — you must `Move()` or explicitly deep-`CopyFrom()` with an allocator. Storing a string requires deciding between a *copy* (`SetString(s, len, allocator)`) and a *reference* (`SetString(StringRef(s))`), and getting this wrong is the most common source of dangling-pointer bugs.
- **In-situ parsing.** `ParseInsitu` mutates the source buffer in place (decoding strings over the original bytes) to avoid copies, trading a mutable, consumed input buffer for lower memory traffic.
- **Encoding as a template parameter.** Streams and documents are parameterized on encoding (`UTF8`, `UTF16`, `UTF32`, LE/BE), so transcoding — e.g. reading UTF-8 into a UTF-16 DOM — happens during parse without a separate pass.
- **Error reporting.** No exceptions by default. `Parse` sets a `ParseErrorCode` you must check via `HasParseError()`/`GetParseError()`; the offset is a byte position, not line/column.

v1.1 (2016) added JSON Pointer, JSON Schema validation, and relaxed-syntax parsing (comments, trailing commas, NaN/Infinity)[^3]. Those remain the newest features; nothing beyond them has been released.

## Production Notes

- **You are pinning a 2016 release.** Because v1.1.0 is the only modern tag, most package managers ship it, but distributions and vendored copies diverge — some pull a specific `master` commit to get post-1.1 bug fixes. Decide deliberately whether to vendor a known `master` SHA or accept the last tag, and record it; "we use RapidJSON" is not a reproducible statement.
- **CVE history.** Older versions carried parsing bugs (e.g. integer/underflow issues in number parsing fixed on `master` after 1.1.0). If you depend on the released tarball you may be missing fixes that only exist on `master`. Audit against your source, not the version string.
- **Allocator lifetime footguns.** A `Value` referencing another document's allocator, or a `StringRef` to a buffer that outlives its scope, compiles cleanly and crashes later. Prefer copying semantics until profiling proves you need references. Under ASan these bugs surface quickly; in release builds they are silent.
- **`kParseFullPrecisionFlag`.** Default number parsing trades a little floating-point accuracy for speed. If you round-trip doubles and need exact reproduction, enable full-precision parsing explicitly — it is off by default.
- **SSE flags must match your target.** `RAPIDJSON_SSE2` / `RAPIDJSON_SSE42` are opt-in macros. Enabling SSE4.2 in a build that runs on hardware without it produces illegal-instruction crashes; leave them off unless you control the deployment target.
- **Thread safety.** Documents and allocators are not thread-safe; a `Document` is single-writer. Sharing one across threads without external synchronization is undefined.
- **Compiler currency.** The codebase predates modern C++ defaults and emits warnings under strict flags on recent GCC/Clang/MSVC. Expect to suppress or patch warnings in `-Werror` builds.

## When to Use / When Not

**Use when:**
- You need maximum parse/serialize throughput in C++ and are comfortable managing allocation lifetimes.
- You want a header-only, dependency-free library that drops into any build.
- You need SAX-style streaming for very large documents that should not be fully materialized.
- You require built-in UTF-16/UTF-32 transcoding or JSON Schema validation.

**Avoid when:**
- You want an ergonomic, modern-C++ API with value semantics and no allocator bookkeeping — the mental overhead is real.
- You need a library under active release-level maintenance with a predictable security-response cadence.
- Your workload is small-config parsing where readability matters more than nanoseconds.
- You want compile-time struct (de)serialization; RapidJSON has no reflection-based mapping.

## Alternatives

- nlohmann/json — the ergonomic default; STL-native, value semantics, far nicer API, slower and heavier. Use when developer time matters more than throughput.
- simdjson/simdjson — SIMD-first parser that is faster for read-only/on-demand parsing of large documents. Use when you parse huge JSON and rarely mutate it.
- Tencent/RapidYAML-style — for YAML, not applicable; noted only to avoid confusion.
- open-source-parsers/jsoncpp — older, mutation-friendly DOM library. Use when you need a stable, simple API and do not care about speed.
- google/flatbuffers or protocolbuffers/protobuf — use instead of JSON entirely when you control both ends and want schema'd binary encoding.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2015-04 | First stable release under Tencent[^2]. |
| 1.1.0 | 2016-08-25 | JSON Pointer, JSON Schema, relaxed syntax, `Value` reduced to 16 bytes[^3]. |
| master | 2025-02 (last push)[^4] | Ongoing bug fixes, no new tagged release. |

## References

[^1]: The project code is MIT-licensed per its README and LICENSE file. GitHub's license detector reports `NOASSERTION` because the repository also bundles third-party components under different terms (e.g. `msinttypes` under BSD and JSON-checker test data under the JSON license). https://github.com/Tencent/rapidjson/blob/master/license.txt
[^2]: RapidJSON README and copyright — "Copyright (C) 2015 THL A29 Limited, a Tencent company, and Milo Yip." https://github.com/Tencent/rapidjson
[^3]: RapidJSON v1.1.0 release / "Highlights in v1.1 (2016-8-25)." https://github.com/Tencent/rapidjson/releases/tag/v1.1.0
[^4]: GitHub repository metadata, `pushed_at` 2025-02-05, last tagged release v1.1.0 (2016). https://github.com/Tencent/rapidjson

## Tags

cpp, json, parser, serialization, header-only, sax, dom, performance, unicode, json-schema, tencent
