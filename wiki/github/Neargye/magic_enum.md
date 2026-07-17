# Neargye/magic_enum

> Compile-time reflection for C++ enums — name, value, iteration, and string round-trips with no macros and no code generation.

[GitHub repo](https://github.com/Neargye/magic_enum) ·
[License: MIT](https://github.com/Neargye/magic_enum/blob/master/LICENSE)

## Overview

`magic_enum` is a header-only C++17 library that gives you the one thing the language itself refuses to: the ability to convert an enum value to its identifier string (and back) without hand-writing a lookup table. `magic_enum::enum_name(Color::RED)` returns `"RED"`; `enum_cast<Color>("RED")` returns the value; `enum_values<Color>()` yields the whole sequence — all `constexpr`, all without a single macro on the enum declaration[^1]. First published in 2019, it has become the default answer to "how do I stringify a C++ enum" and ships in vcpkg, Conan, Meson wrapdb, and Build2.

The defining tension is that this is not real reflection. C++17 has no language mechanism to enumerate an enum's members, so `magic_enum` extracts names by parsing the compiler's own `__PRETTY_FUNCTION__` / `__FUNCSIG__` diagnostic strings at compile time. That trick is clever and portable across the major compilers, but it is fundamentally a hack on unspecified behavior, and it carries two hard constraints: enum values must fall inside a bounded integer range (default `[-128, 128]`), and reflecting large enums or wide ranges inflates compile time and memory[^2]. The maintainer is explicit that the library "does not pretend to be a silver bullet" and was designed for small enums[^1].

The approach is also a stopgap. C++26's static reflection (P2996) will provide first-class enum introspection, at which point the compiler-string parsing that powers `magic_enum` becomes unnecessary. Until compilers ship that, this library remains the pragmatic choice for the C++17/20 era.

## Getting Started

Header-only: drop `include/magic_enum/magic_enum.hpp` into your project, or install via a package manager.

```bash
# vcpkg
vcpkg install magic-enum
# Conan
conan install --requires=magic_enum/0.9.7
```

```cpp
#include <magic_enum/magic_enum.hpp>
#include <iostream>

enum class Color { RED = -10, BLUE = 0, GREEN = 10 };

int main() {
  Color c = Color::GREEN;
  std::cout << magic_enum::enum_name(c) << '\n';          // "GREEN"

  auto parsed = magic_enum::enum_cast<Color>("BLUE");
  if (parsed.has_value()) { /* parsed.value() == Color::BLUE */ }

  constexpr auto count = magic_enum::enum_count<Color>();  // 3
  for (Color v : magic_enum::enum_values<Color>()) {
    std::cout << magic_enum::enum_name(v) << ' ';          // RED BLUE GREEN
  }
}
```

## Architecture / How It Works

There is no macro, no build step, and no annotation on your enum. The entire mechanism is template metaprogramming over compiler diagnostic strings.

For a candidate value `V` of enum type `E`, `magic_enum` instantiates a template function whose signature the compiler renders into a string — `__PRETTY_FUNCTION__` on GCC/Clang, `__FUNCSIG__` on MSVC. When `V` corresponds to a real enumerator, that string contains the enumerator's identifier (e.g. `...E::RED...`); when it does not, the compiler emits a cast expression or a numeric literal instead. `magic_enum` parses the string at compile time to decide which candidates are valid and to slice out the names[^2].

Because it must probe candidate integers one at a time, the library scans a fixed range: `MAGIC_ENUM_RANGE_MIN` to `MAGIC_ENUM_RANGE_MAX`, default `-128` to `128`. Every enum reflected costs one template instantiation per candidate in that range. This is the origin of both its main limitation (values outside the range are invisible) and its compile-time cost. The range is configurable globally via those macros, or per-enum by specializing `magic_enum::customize::enum_range<E>` — the same specialization point used to flag bitwise-flag enums with `is_flags = true`.

Beyond the core name/value/cast primitives, the library layers on: `enum_flags_name`/`enum_flags_cast` for bit-flag enums, `enum_switch`/`enum_for_each` to recover a runtime value as a `constexpr` constant inside a lambda, `enum_fuse` for multi-dimensional switch/case, `containers::array`/`bitset`/`set` (enum-keyed containers), out-of-the-box `operator<<`/`>>` and bitwise operators (opt-in via namespaces so they never leak), and a UB-free `underlying_type`. A separate single-value form, `enum_name<Color::BLUE>()`, takes the value as a template argument and thereby sidesteps the range limitation entirely — at the cost of needing the value at compile time. C++20 modules are supported (`import magic_enum;`) as an alternative to header inclusion, requiring CMake 3.28+.

## Production Notes

**Range is a silent correctness footgun, not just a perf knob.** Any enumerator outside `[-128, 128]` — error codes, hashed IDs, `1 << 20` flag bits — returns an empty name or a nullopt cast with no warning. Widen the range with `customize::enum_range<E>` per enum rather than bumping the global bounds, because the global setting multiplies compile cost across every reflected enum.

**Compile time and memory scale with range × number of reflected enums.** A wide `MAGIC_ENUM_RANGE_MAX` on a translation unit that reflects many enums can turn into minutes of build time and gigabytes of compiler memory; this has been the recurring complaint in the issue tracker. Keep ranges tight, prefer the `enum_name<value>()` static-storage form in hot compile paths, and reflect enums in as few translation units as practical.

**It leans on unspecified compiler behavior.** Names are recovered from diagnostic-string formats that no standard guarantees. New compiler versions occasionally change those formats and break extraction until the library patches its parser; MSVC in particular has historically been the fragile target. Pin a `magic_enum` version you have tested against your exact compiler and treat compiler upgrades as something to re-verify.

**Aliased enumerators are ambiguous.** When two enumerators share the same underlying value, only one name is recoverable, and which one you get is not something to rely on. Enums that use value aliases for readability will not round-trip cleanly.

**Module vs header is an ODR trap.** Do not mix `#include <magic_enum/...>` and `import magic_enum;` in the same link unit — the README calls this out explicitly as an ODR violation[^1]. Pick one integration mode per binary.

**Not for reflection-heavy designs.** If you are reflecting hundreds of large enums, or enums whose values are genuinely arbitrary integers, this is the wrong tool; a macro-based or code-generated approach will compile faster and cover the full value space.

## When to Use / When Not

**Use when:**
- You have small, dense enums and want to-string / from-string without maintaining a switch or lookup table.
- You cannot or will not annotate the enum declaration with a macro (third-party enums, generated code).
- You need `constexpr` enum name/value/iteration in C++17 or C++20 today, before C++26 reflection is available.
- Logging, serialization, config parsing, or CLI mapping where the enum set is human-sized.

**Avoid when:**
- Enumerators live outside a small integer range (error codes, bitmasks with high bits) and you don't want per-enum range tuning.
- You reflect many or very large enums and compile time is a constraint.
- You need behavior guaranteed by the standard — this depends on compiler diagnostic-string internals.
- You are on a compiler older than Clang 5 / GCC 9 / MSVC 2017 (14.11) / Xcode 10, which are the stated minimums[^1].

## Alternatives

- aantron/better-enums — macro-declared enums (`BETTER_ENUM(...)`); no value-range limit and works back to C++11, at the cost of changing how you declare the enum.
- boostorg/describe — annotation-macro reflection covering enums, structs, and members; use when you want one reflection scheme across more than just enums.
- quicknir/wise_enum — macro-based reflective enum with no range scanning; use when `magic_enum`'s compile cost or range bound is unacceptable and you can define enums via a macro.
- P2996 static reflection (C++26, standard) — the eventual language-level replacement; use once your compiler supports it and you can require C++26.
- fmt / hand-written `to_string` switch — for a single small enum, a plain switch is zero-dependency and fully in your control.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-03-30 | Repository created; core `enum_name` / `enum_cast` / `enum_values` for C++17[^3]. |
| 0.7.x era | ~2020–2021 | Flag-enum support, `customize::enum_range`, case-insensitive `enum_cast`. |
| 0.8.x era | ~2022 | `enum_switch` / `enum_for_each` / `enum_fuse`; broader compiler coverage. |
| 0.9.x era | ~2023–2024 | `containers::array`/`bitset`/`set`; C++20 modules (`import magic_enum;`). |
| maintained | 2026-07 | Actively maintained; last push 2026-07-11, ~6.1k stars, MIT[^3]. |

## References

[^1]: magic_enum README — features, integration, compiler compatibility, and remarks. https://github.com/Neargye/magic_enum/blob/master/README.md
[^2]: magic_enum limitations documentation — value range, compile-time cost, and how names are recovered. https://github.com/Neargye/magic_enum/blob/master/doc/limitations.md
[^3]: GitHub repository metadata (Neargye/magic_enum) — created 2019-03-30, MIT license, star/fork counts and last-push date as of 2026-07. https://github.com/Neargye/magic_enum

## Tags

cpp, c-plus-plus-17, header-only, reflection, metaprogramming, enum, enum-to-string, serialization, compile-time, no-dependencies
