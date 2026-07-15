# fmtlib/fmt

> A fast, type-safe C++ formatting library — and the reference implementation that became C++20 `std::format`.

[GitHub repo](https://github.com/fmtlib/fmt) ·
[Official website](https://fmt.dev) ·
[License: MIT](https://github.com/fmtlib/fmt/blob/master/LICENSE)

## Overview

{fmt} is a C++ formatting library that provides an alternative to C `stdio`
(`printf`) and C++ `iostreams`. It offers a Python-style format-string syntax
(`fmt::format("{}", x)`), full compile-time type safety, and IEEE 754
floating-point output with correct rounding and round-trip guarantees. Started
by Victor Zverovich around 2012 under the name *cppformat*, it was renamed
{fmt} and has been maintained continuously since[^1].

Its defining significance is standardization: the library is the basis of the
C++20 `std::format` facility (proposal P0645) and the C++23 `std::print`
(P2093), both authored by the same maintainer[^2]. This makes {fmt} unusual
among third-party libraries — it is simultaneously a dependency you add today
*and* a preview of what the standard library will eventually give you. The
practical tension that follows: as `std::format`/`std::print` support matures
in vendor standard libraries, the question "why not just use `std`?" gets
louder, while {fmt} continues to ship features (color, terminal styling, richer
ranges/chrono formatting, older-compiler support) that the standard lags on.

{fmt} is pervasive in production C++. It is vendored or depended on by spdlog,
Folly, ClickHouse, MongoDB, PyTorch, Envoy, Ceph, and Scylla among many
others[^1]. For a large fraction of C++ codebases it arrives transitively
through a logging library rather than by direct choice.

## Getting Started

With CMake `FetchContent` (no system install needed):

```cmake
include(FetchContent)
FetchContent_Declare(fmt
  GIT_REPOSITORY https://github.com/fmtlib/fmt.git
  GIT_TAG        11.1.0)   # pin a release tag
FetchContent_MakeAvailable(fmt)
target_link_libraries(my_app PRIVATE fmt::fmt)
```

A minimal program:

```cpp
#include <fmt/base.h>     // lightweight core; formerly fmt/core.h

int main() {
  fmt::print("Hello, {}!\n", "world");
  std::string s = fmt::format("The answer is {}.", 42);
  fmt::print("{}\n", s);
}
```

Runtime (non-constant) format strings must be wrapped, since literals are
checked at compile time by default:

```cpp
std::string tmpl = load_from_config();
fmt::print(fmt::runtime(tmpl), 42);   // opt out of compile-time checking
```

## Architecture / How It Works

The library is organized into layered headers so you pay for only what you
include — the biggest lever on its compile-time cost:

- **`fmt/base.h`** (renamed from `fmt/core.h` in the v11 series) — the core
  formatting API with minimal dependencies. Sufficient for `fmt::print` /
  `fmt::format` on built-in types.
- **`fmt/format.h`** + **`fmt/format-inl.h`** — the full formatter, including
  the floating-point machinery. These three files are the "minimum
  configuration" the project advertises.
- **`fmt/chrono.h`, `fmt/ranges.h`, `fmt/color.h`, `fmt/os.h`, `fmt/std.h`,
  `fmt/printf.h`** — opt-in extensions for durations/time points, containers,
  terminal styling, file output, standard-library types, and a POSIX-style
  `printf`.

Formatting dispatches through the `fmt::formatter<T>` trait; user-defined types
are supported by specializing it. Arguments are type-erased into a compact
`format_args` structure, which is what keeps per-call binary size close to
`printf` and avoids the template bloat characteristic of iostreams.

Floating-point formatting uses the **Dragonbox** algorithm for the shortest
round-trippable decimal representation with correct rounding[^3]. This is the
source of {fmt}'s large speed advantage over `sprintf`/`ostringstream` on
`float`/`double`.

**Compile-time checking**: since the 8.x series, string literals passed to
`fmt::format`/`fmt::print` are validated at compile time using `consteval` on
C++20 (and via `FMT_STRING` on older standards). An invalid specifier such as
`fmt::format("{:d}", "not a number")` becomes a compilation error rather than a
runtime throw. Versioning uses an inline namespace (`fmt::v11`) so symbols carry
the ABI version.

## Production Notes

**Header-only vs compiled.** {fmt} defaults to a compiled library; header-only
mode is opt-in via `FMT_HEADER_ONLY`. Header-only is convenient but pushes the
floating-point and parsing code into every translation unit that formats,
materially increasing compile times on large projects. Prefer the compiled
`fmt::fmt` target and include `fmt/base.h` (not `fmt/format.h`) where you only
need the core API.

**Version pinning and the diamond problem.** Because {fmt} is so widely
vendored, a non-trivial C++ build can pull in two different {fmt} versions —
e.g. one bundled inside spdlog and another linked directly. The inline-namespace
versioning prevents silent ABI mismatches, but header-only builds mixing
versions can still hit ODR issues. Pin one version and, where possible, make
spdlog use your external {fmt} (`SPDLOG_FMT_EXTERNAL`).

**Compile-time-check upgrade footgun.** Teams upgrading from pre-8.0 releases
are frequently surprised when previously-fine code fails to compile because a
format string is no longer a constant expression (e.g. it comes from a
variable). The fix is `fmt::runtime(str)` — but it must be applied deliberately;
it is the most common source of post-upgrade build breaks.

**`std::format` migration is not a drop-in swap.** {fmt} is a superset. Moving
to the standard library loses `fmt/color.h`, some ranges/chrono formatting
conveniences, and support for compilers whose standard library ships an
incomplete or missing `<format>`/`<print>`. Keep {fmt} if you need those or must
support older toolchains; consider `std` once your minimum compiler baseline
covers it.

**Locale.** Formatting is locale-independent by default, which is usually what
you want for logs, wire formats, and reproducible output. Locale-aware output is
explicitly opt-in (the `L` specifier). This is the opposite default from
iostreams and is a deliberate correctness choice.

## When to Use / When Not

**Use when:**
- You want fast, type-safe formatting with a readable `{}` syntax today, across
  a range of compilers including older ones.
- You need correct, fast float formatting or locale-independent output.
- You want terminal color/styling, or rich container and chrono formatting that
  the standard library does not yet cover.
- You're already pulling it in transitively (spdlog, Folly) and want to use it
  directly rather than add another formatter.

**Avoid when:**
- Your minimum toolchain already ships a complete `<format>`/`<print>` and you
  need none of the extensions — the standard library removes a dependency.
- You cannot tolerate the compile-time cost and are tempted by header-only mode
  in a very large codebase without a compiled target.
- You need a full logging solution (levels, sinks, async) — reach for a logging
  library that is built on top of it instead.

## Alternatives

- C++ Standard Library `std::format` / `std::print` — the standardized subset of
  this very library; use it when your compiler baseline supports it and you need
  no {fmt}-only extensions.
- gabime/spdlog — logging library built on {fmt}; use when you need sinks,
  levels, and async logging rather than raw formatting.
- abseil/abseil-cpp (`absl::StrFormat`) — type-safe printf-style formatting; use
  when you already depend on Abseil.
- boostorg/format — older `operator%` formatting; markedly slower and larger, so
  avoid for new code unless already committed to Boost.
- c42f/tinyformat — tiny header-only printf-style formatter; use when a minimal
  single-header dependency matters more than speed or features.

## History

| Version | Date | Notes |
|---------|------|-------|
| (cppformat) | 2012 | Repository created; original name before the rename[^1]. |
| 3.0 | 2016 | Renamed to {fmt}. |
| 5.0 | 2018 | Named-argument and API maturation. |
| 6.0 | 2019 | `fmt/core.h` introduced as the lightweight core header. |
| 7.0 | 2020 | Basis of C++20 `std::format` (P0645) adopted into the standard[^2]. |
| 8.0 | 2021 | Compile-time format-string checks on by default (C++20 `consteval`). |
| 9.0 | 2022 | Expanded ranges/chrono formatting. |
| 10.0 | 2023 | C++23 `std::print` (P2093) lands in the standard[^2]. |
| 11.0 | 2024 | `fmt/core.h` renamed to `fmt/base.h`. |
| 12.x | 2025 | Current release series (benchmarks cite 12.1)[^1]. |

## References

[^1]: {fmt} README and project documentation. https://github.com/fmtlib/fmt and https://fmt.dev
[^2]: `std::format` (C++20, P0645) and `std::print` (C++23, P2093), authored by the {fmt} maintainer. https://en.cppreference.com/w/cpp/utility/format and https://en.cppreference.com/w/cpp/io/print
[^3]: Junekey Jeon, "Dragonbox" shortest-round-trip float-to-string algorithm used by {fmt}. https://github.com/jk-jeon/dragonbox

## Tags

cpp, c-plus-plus, formatting, text-formatting, std-format, printf, floating-point, unicode, header-library, cross-platform, performance
