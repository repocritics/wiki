# Neargye/nameof

> Header-only C++17 library that returns the source-code name of a variable, type, function, or enum value as a compile-time string — by parsing the compiler's own signature strings.

[GitHub repo](https://github.com/Neargye/nameof) ·
No official website ·
[License: MIT](https://github.com/Neargye/nameof/blob/master/LICENSE)

## Overview

`nameof` solves a problem C++ did not natively have an answer to before the C++26 reflection work: turning a program symbol into its textual name without hand-writing string tables. `NAMEOF(x)` yields `"x"`, `NAMEOF_ENUM(color)` yields `"RED"`, and `NAMEOF_TYPE(T)` yields `"my::detail::SomeClass<int>"` — all as `constexpr` `string_view`, evaluated at compile time with no runtime cost and, in the common cases, no RTTI. It is maintained by Daniil Goncharov (Neargye), the same author as `magic_enum`, and the two libraries share the same underlying trick.[^1]

The defining tension is that C++17 has no reflection, so `nameof` manufactures it. Type and enum names are recovered by taking a templated function, reading the compiler-generated signature string (`__PRETTY_FUNCTION__` on GCC/Clang, `__FUNCSIG__` on MSVC), and slicing out the substring that names the template argument.[^2] This works remarkably well in practice, but it means the library sits on undocumented, compiler-specific string formats. It is a pragmatic hack that most projects will never outgrow, but one whose limitations are real and worth reading before adoption.

## Getting Started

`nameof` is a single header. Copy `include/nameof.hpp` into your project, or pull it via a package manager — it is in vcpkg, Conan Center, and the Arch AUR.[^3]

```bash
# vcpkg
vcpkg install nameof
# Conan
conan install --requires=nameof/0.10.4
```

```cpp
#include <nameof.hpp>
#include <iostream>

enum class Color { RED = 1, BLUE = 2, GREEN = 4 };

int main() {
  int somevar = 0;
  std::cout << NAMEOF(somevar) << '\n';          // "somevar"
  std::cout << NAMEOF_ENUM(Color::GREEN) << '\n'; // "GREEN"

  const std::vector<int>& v = {};
  std::cout << NAMEOF_TYPE_EXPR(v) << '\n';        // "std::vector<int>" (spelling varies)
  return 0;
}
```

## Architecture / How It Works

There are two distinct mechanisms behind the macro family:

- **Token-based names** — `NAMEOF(person.address.zip_code)` is fundamentally a preprocessor operation. The macro stringizes the expression, then a `constexpr` pass trims it down to the trailing identifier (`"zip_code"`). `NAMEOF_FULL` and `NAMEOF_RAW` expose earlier stages of that trimming. This path is fully portable because it never touches the compiler's internals — it only reads what you literally typed.

- **Signature-parsing names** — `NAMEOF_TYPE`, `NAMEOF_ENUM`, and `NAMEOF_MEMBER` cannot be derived from the source token, so they use the signature trick: instantiate a helper template like `nameof_type<T>()`, read `__PRETTY_FUNCTION__`/`__FUNCSIG__` inside it, and extract the argument spelling by offset from known delimiter characters. The offsets are calibrated per compiler.[^2]

Enum reflection is the heaviest case. `NAMEOF_ENUM(value)` does not know `value` at compile time, so it brute-forces: for every integer in a configured `enum_range` (default roughly `[-128, 128]`), it instantiates the signature helper, checks whether that value corresponds to a named enumerator, and builds a lookup. At runtime it indexes into that table. This is why the README steers you to `NAMEOF_ENUM_CONST(Color::GREEN)` when the value is known at compile time — that variant instantiates exactly one helper and sidesteps both the range limit and most of the compile-time cost.

RTTI variants (`NAMEOF_TYPE_RTTI`, for polymorphic `*ptr`) fall back to `typeid(...).name()` plus demangling, because the dynamic type is only knowable at runtime. These are the only functions that require RTTI to be enabled.

## Production Notes

- **Compile time and memory scale with enum range.** Every widening of `enum_range` multiplies template instantiations for `NAMEOF_ENUM`. Large sparse enums (e.g. Windows error codes, protocol constants above 128) either need `magic_enum`-style range customization or won't be found at all. For known constants, prefer `NAMEOF_ENUM_CONST`.
- **Type-name strings are not portable.** The exact spelling from `NAMEOF_TYPE` differs across compilers and versions — MSVC may emit `class` / `struct` prefixes, GCC and Clang differ on spacing and `std::__cxx11` inline namespaces. Do not use these strings as stable keys across toolchains, in serialization formats, or in wire protocols. They are for logging and diagnostics.
- **It rides undocumented compiler behavior.** Because names come from parsing `__PRETTY_FUNCTION__`/`__FUNCSIG__`, a new compiler release can change the format and break extraction until the library is patched. Pin a `nameof` version and test on your exact toolchain.
- **Compiler floor.** Requires C++17 with reasonably modern GCC, Clang, and MSVC. `NAMEOF_ENUM` in particular has historically been the first thing to break on bleeding-edge or exotic compilers.
- **RTTI dependency is isolated but real.** Only the `_RTTI` variants need RTTI; the rest work with `-fno-rtti`. If you disable RTTI globally, avoid the polymorphic-type functions.
- **Header cost.** As a single header pulled into many translation units, it adds to compile time; it is lighter than `magic_enum` if you only need name extraction, heavier than hand-written tables.

## When to Use / When Not

**Use when:**
- You want readable enum-to-string, type names, or variable names for logging, assertions, and debug output without maintaining string tables.
- You need a zero-dependency, single-header drop-in and can pin your compiler.
- Your enums are small and dense, or you can use the compile-time-constant variants.

**Avoid when:**
- You need type-name strings that are byte-identical across compilers (serialization keys, stable IDs) — the spellings differ.
- Your enums are large, sparse, or exceed the default range and compile-time budget matters.
- You want full runtime reflection (iterate members, set fields by name, construct by type name) — this library only extracts names.
- You are already on a toolchain with C++26 static reflection, which supersedes the trick.

## Alternatives

- Neargye/magic_enum — same author, focused entirely on enum reflection (iteration, count, bitwise flags, range customization). Use it when enums are the whole job.
- boostorg/pfr — macro-free reflection over aggregate struct fields (count, access, tie). Use it when you need to iterate members, not just read one name.
- rttrorg/rttr — full runtime reflection with registration. Use it when you need to enumerate and invoke members/methods by name at runtime.
- getml/reflect-cpp — struct reflection aimed at serialization (JSON, etc.). Use it when name extraction is really a serialization need.
- C++26 static reflection (P2996) — the language feature that will eventually replace signature-parsing hacks. Use it when your compiler ships it.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-03-17 | Repository created; `NAMEOF` for variable/function names.[^4] |
| 0.9.x | 2019–2020 | Enum and type name support matured. |
| 0.10.x | 2020–2026 | Added `NAMEOF_ENUM_FLAG`, `NAMEOF_MEMBER`, `NAMEOF_POINTER`, short/full type variants; current series (latest 0.10.4). |

Exact per-release dates within the 0.9/0.10 series are not verified here; consult the GitHub releases page for authoritative tags.

## References

[^1]: Neargye/nameof repository and README, MIT License. https://github.com/Neargye/nameof
[^2]: The signature-parsing technique (`__PRETTY_FUNCTION__` / `__FUNCSIG__`) is shared with `magic_enum`; see its documentation for a fuller explanation. https://github.com/Neargye/magic_enum
[^3]: Integration options (vcpkg, Conan Center, AUR, CPM) per the project README. https://github.com/Neargye/nameof#integration
[^4]: GitHub repository metadata (created 2018-03-17). https://github.com/Neargye/nameof

## Tags

cpp, cpp17, header-only, reflection, enum-to-string, metaprogramming, compile-time, no-dependencies, single-header, serialization
