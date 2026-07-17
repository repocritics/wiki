# USCiLab/cereal

> A header-only C++11 serialization library that turns arbitrary types into binary, XML, or JSON via a single `serialize` function — no schema, no codegen, no dependencies.

[GitHub repo](https://github.com/USCiLab/cereal) ·
[Official website](https://uscilab.github.io/cereal) ·
[License: BSD-3-Clause](https://opensource.org/licenses/BSD-3-Clause)

## Overview

cereal is a header-only serialization library for C++11, developed out of USC's iLab and first released as 1.0 in 2014[^1]. It takes C++ objects and reversibly encodes them into one of several formats — a compact binary representation, a portable (endian-normalized) binary, XML, or JSON — through the same interface. You describe a type once by writing a `serialize` (or split `save`/`load`) function that lists its members, and cereal handles the rest, including the C++ standard library containers and smart pointers out of the box.

Its defining design choice, and its defining tradeoff, is that the C++ type *is* the schema. There is no `.proto` file, no code generator, and no intermediate description language: serialization logic lives inside your own headers as a template function. This makes cereal ergonomic and dependency-free for C++-to-C++ persistence, and it is the modern successor to Boost.Serialization in that role[^2]. But it also means cereal offers none of the cross-language interop, wire-format stability, or forward/backward compatibility guarantees that schema-first systems like Protocol Buffers provide. cereal serializes C++ objects for other C++ programs — nothing else.

The project is stable and widely used but low-velocity. Releases are infrequent (the 1.3.x line has spanned years), and the issue tracker carries several hundred open items[^3], which reflects a mature codebase with limited active maintainer bandwidth rather than instability. For its intended niche it is effectively feature-complete.

## Getting Started

cereal is header-only. Vendor the `include/` tree, or install through a package manager:

```bash
# vcpkg
vcpkg install cereal
# Conan
conan install --requires=cereal/1.3.2
# Debian/Ubuntu
apt-get install libcereal-dev
# macOS
brew install cereal
```

```cpp
#include <cereal/archives/json.hpp>
#include <cereal/types/vector.hpp>
#include <iostream>
#include <sstream>

struct Point {
  double x, y;
  template <class Archive>
  void serialize(Archive& ar) { ar(CEREAL_NVP(x), CEREAL_NVP(y)); }
};

int main() {
  std::stringstream ss;
  {
    cereal::JSONOutputArchive out(ss);
    std::vector<Point> pts{{1, 2}, {3, 4}};
    out(cereal::make_nvp("points", pts));
  } // archive MUST leave scope here — its destructor flushes closing tokens

  std::vector<Point> loaded;
  {
    cereal::JSONInputArchive in(ss);
    in(loaded);
  }
  std::cout << loaded.size() << "\n"; // 2
}
```

## Architecture / How It Works

cereal is organized around two orthogonal concepts: **archives** and **serialization functions**. An archive (`BinaryOutputArchive`, `PortableBinaryInputArchive`, `XMLOutputArchive`, `JSONInputArchive`, etc.) owns a stream and knows how to read or write primitives. A serialization function, defined per type, decomposes an object into a sequence of `ar(member1, member2, ...)` calls. Because both are templates, the same type description works with every archive format — dispatch is resolved entirely at compile time.

You can express serialization four ways, resolved by SFINAE: a member `serialize`, a member split `save`/`load`, or non-member (free function) equivalents for types you cannot modify. cereal detects which form exists and errors at compile time if a type is ambiguous or unserializable. Standard-library support ships as opt-in headers under `cereal/types/` (`vector.hpp`, `memory.hpp`, `unordered_map.hpp`, and so on) — you include only what you use.

Polymorphism is handled through a static type registry. `CEREAL_REGISTER_TYPE(Derived)` records a base→derived mapping so that a `shared_ptr<Base>` round-trips to the correct concrete type. Under the hood this relies on static initialization to populate a global table, keyed by a type name string[^4].

Versioning is opt-in: `CEREAL_CLASS_VERSION(T, N)` switches a type's serialize function to a two-argument form receiving the version integer, letting you branch load logic across format revisions. Without it, cereal has no notion of a field being added or removed — the read side must match the write side exactly.

The XML and JSON archives bundle rapidxml and RapidJSON respectively inside `cereal/external/`, which is why the library reports zero external dependencies. Name-value pairs (`CEREAL_NVP`, `make_nvp`) attach human-readable keys, which the text archives emit and the binary archives silently ignore.

## Production Notes

- **Binary archives are not portable.** `BinaryOutputArchive` writes native-endian, native-size representations directly. Data written on one architecture will not correctly load on another with different endianness or type widths. Use `PortableBinaryArchive` for cross-platform binary, and be aware it still assumes matching floating-point representation.
- **Archives must be destroyed before you read the stream.** The JSON and XML output archives write closing tokens in their destructor. Reading a stream while the archive is still in scope yields truncated or invalid output — hence the explicit `{}` blocks in the example above. This is the single most common cereal bug report.
- **Polymorphic registration can be silently dropped by the linker.** Because `CEREAL_REGISTER_TYPE` depends on static initialization, when a registered type lives in a static library or a translation unit whose symbols are otherwise unreferenced, the linker may strip the registration, producing a runtime "trying to save/load an unregistered polymorphic type" exception. The documented mitigation is `CEREAL_REGISTER_DYNAMIC_INIT` plus a forced link reference[^4].
- **No schema evolution.** cereal has no optional or defaulted fields. Adding, removing, or reordering members breaks compatibility with previously written data unless you manually gate every change behind class versioning. It is unsuitable for long-lived on-disk formats that must evolve, or for message passing between independently deployed services.
- **Compile-time and binary-size cost.** Being deeply template-based and header-only, cereal inflates compile times, especially in projects with many serialized types or heavy polymorphism. Expect this to show up in incremental build latency.
- **Thread safety.** Individual archives are not thread-safe; each thread needs its own archive over its own stream. The polymorphic type registry is populated at static-init time and read-only thereafter.

## When to Use / When Not

**Use when:**
- You need to persist or transfer C++ objects between C++ programs and control both ends.
- You want serialization logic that lives in your headers with no codegen step and no runtime dependencies.
- You value being able to switch between binary, JSON, and XML from one type description.
- You are migrating off Boost.Serialization and want a lighter, more modern equivalent.

**Avoid when:**
- You need cross-language interop or a stable wire format — reach for a schema-first system.
- Your format must evolve over years with forward/backward compatibility across versions.
- You need zero-copy or memory-mapped reads of large payloads.
- You want maximum binary throughput/size efficiency for a hot path.

## Alternatives

- protocolbuffers/protobuf — use instead when you need cross-language interop, schema evolution, and a documented wire format.
- google/flatbuffers — use instead when you need zero-copy access to serialized data without a parse step.
- fraillt/bitsery — use instead when you want the same header-only C++ ergonomics but faster and smaller binary output.
- nlohmann/json — use instead when you only need human-readable JSON and not a multi-format archive abstraction.
- boostorg/serialization — use instead when you are already committed to Boost and want its broader ecosystem and archive types.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2014 | First public release; archives, STL support, polymorphism[^1]. |
| 1.1.0 | 2015 | API refinements, additional type support. |
| 1.2.0 | 2016 | Bug fixes, expanded standard-library coverage. |
| 1.3.0 | 2017 | Maintenance and portability improvements. |
| 1.3.1 | 2019 | Bug-fix release. |
| 1.3.2 | 2022 | Latest tagged release; compiler-compatibility and bug fixes. |

## References

[^1]: cereal official documentation and release notes. https://uscilab.github.io/cereal
[^2]: cereal transition guide from Boost.Serialization. https://uscilab.github.io/cereal/transition_from_boost.html
[^3]: USCiLab/cereal issue tracker (open issue count ~340 as of 2026-03). https://github.com/USCiLab/cereal/issues
[^4]: cereal documentation, "Polymorphism". https://uscilab.github.io/cereal/polymorphism.html

## Tags

cpp, c-plus-plus, serialization, header-only, binary-format, json, xml, cpp11, data-persistence, boost-serialization-alternative
