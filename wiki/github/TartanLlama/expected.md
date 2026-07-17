# TartanLlama/expected

> Single-header C++11/14/17 implementation of `std::expected` with functional-style monadic extensions, predating the standard.

[GitHub repo](https://github.com/TartanLlama/expected) ·
[Official website](https://tl.tartanllama.xyz) ·
[License: CC0-1.0](https://github.com/TartanLlama/expected/blob/master/COPYING)

## Overview

`tl::expected` is a header-only backport of the `expected<T, E>` vocabulary type
written by Sy Brand (TartanLlama). It tracks the p0323r3 proposal[^1] — the
interface that eventually became `std::expected` in C++23 — but works all the
way back to C++11. An `expected<T, E>` holds either a value of type `T` or an
"unexpected" error of type `E`, giving you sum-type error handling without
exceptions and without the "valid-but-unspecified" ambiguity of returning a
status code alongside an out-parameter.

The library's reason to exist beyond the raw type is the set of functional
extensions layered on top: `map`, `map_error`, `and_then`, and `or_else`. These
let you chain fallible operations so error-propagation plumbing disappears into
the combinators rather than being interleaved with `if (!x) return x;` checks.
This is the same "railway-oriented" error handling familiar from Rust's `Result`
or Haskell's `Either`.

The defining tension in 2026 is that this library has been overtaken by the
standard it prototyped. `std::expected` shipped in C++23, so for new code on a
modern toolchain the library is redundant. It remains relevant as a drop-in for
codebases stuck on C++11/14/17, and as a reference — but the naming diverges
from the standard in a way that makes migration non-trivial (see Production
Notes). It is licensed CC0-1.0, i.e. public domain, so vendoring the single
header carries no attribution obligation.

## Getting Started

The library is a single header with no dependencies. Copy `tl/expected.hpp` into
your tree, or pull it via a package manager:

```bash
vcpkg install tl-expected          # port name: tl-expected
conan install tl-expected/1.1.0@   # Conan (community package)
```

```cpp
#include <tl/expected.hpp>
#include <string>

tl::expected<int, std::string> parse(const std::string& s) {
    try {
        return std::stoi(s);
    } catch (...) {
        return tl::make_unexpected("not a number");
    }
}

int main() {
    tl::expected<int, std::string> r =
        parse("21").map([](int x) { return x * 2; });

    if (r) {
        return *r;              // 42
    }
    // r.error() holds "not a number"
    return -1;
}
```

`operator*` / `operator->` access the value; `.error()` accesses the error.
Calling `.error()` on a value-holding `expected` (or dereferencing an error-
holding one) is undefined behaviour per the proposal; this implementation turns
it into an assertion failure instead, overridable via `TL_ASSERT`.

## Architecture / How It Works

Internally `expected<T, E>` is a tagged union: storage for a `T` or a
`tl::unexpected<E>`, plus a `bool has_value` discriminant. The engineering
substance is in making that union behave correctly across the full C++11–17
range. The type propagates triviality of copy, move, and destruction from `T`
and `E` by selecting among several conditional base-class specializations
(SFINAE- and tag-dispatched) — so an `expected<int, int>` remains trivially
destructible and constexpr-friendly, while an `expected<std::string, E>` gets a
non-trivial destructor. This is the same layered-base technique the author uses
in the sister library `tl::optional`, and it is what lets the header target
compilers as old as clang 3.5 and gcc 4.8 without the C++17 features (`if
constexpr`, guaranteed copy elision) that would make it simpler.

The functional extensions are thin wrappers:

- **`map(f)`** — if a value is present, returns `expected<U, E>` where `U` is
  `f`'s return type; otherwise forwards the error. Analogous to `transform` in
  the standard.
- **`map_error(f)`** — transforms the error type, leaving the value path
  untouched. Analogous to `transform_error`.
- **`and_then(f)`** — like `map`, but `f` itself returns an `expected`, so the
  result is flattened rather than nested (monadic bind).
- **`or_else(f)`** — invoked on the error path; used for recovery or to raise.

Because the whole thing is header-only and template-heavy, there is no runtime
library and no ABI surface — the cost is entirely at compile time.

## Production Notes

**Naming divergence from `std::expected` is the migration footgun.** The
standard chose `transform` / `transform_error` where this library uses `map` /
`map_error`; `and_then` and `or_else` match. Code written against `tl::expected`
does not compile unchanged against `std::expected` — a mechanical rename is
required, and the two names can coexist confusingly in a codebase mid-migration.
Plan a single sweep rather than a gradual swap.

**It is effectively in low-frequency maintenance.** Development is sporadic:
after v1.1.0 (2023) the project sat idle until v1.2.0 in mid-2025, with v1.3.x
following the same day as a routine housekeeping bump[^2]. The open-issue count
(dozens) reflects accumulated backlog more than active triage. For a frozen
vocabulary type this is acceptable — the interface is stable and the behaviour
matches a shipped standard — but do not expect fast turnaround on edge-case bug
reports.

**`TL_ASSERT` controls contract-violation behaviour.** By default a misuse (e.g.
`.error()` on a value) fires `assert()` from `<cassert>`, which is compiled out
under `NDEBUG`. In release builds this means misuse is silently UB. Define
`TL_ASSERT(cond)` before including the header if you want a hard check in
production. Correct code should never rely on these assertions firing.

**No exceptions required, but conversions can throw.** The type itself is
exception-neutral; however constructing the contained `T`/`E` runs their
constructors, which may throw. `expected` provides the strong-guarantee dance
around assignment where the standard requires it, but the usual caveats about
throwing move constructors apply.

**Reference-type and `void` value support.** `expected<void, E>` is supported
for operations that either succeed or fail with an error and produce no value —
useful for command-style APIs.

## When to Use / When Not

**Use when:**
- You are on C++11/14/17 and want `std::expected`-style error handling now.
- You want the `map`/`and_then` monadic chaining ergonomics specifically.
- You need a zero-dependency, public-domain single header you can vendor freely.
- You are maintaining existing code already built on `tl::expected`.

**Avoid when:**
- You target C++23 or newer — use `std::expected` from the standard library and
  skip the third-party dependency and the naming mismatch.
- You want a maintained, feature-rich error framework with policy hooks and
  no-throw guarantees baked in — Boost.Outcome fits better.
- You need the exact standard names in a codebase you expect to migrate — a
  library whose combinators already match `std::expected` reduces churn.

## Alternatives

- martinmoene/expected-lite — another C++11 single-header `expected` backport; more configuration knobs and closer to standard naming. Use when you want a std-aligned API for easier future migration.
- ned14/outcome (Boost.Outcome) — heavier `result`/`outcome` types with explicit no-throw policies and richer error plumbing. Use when error handling is a first-class subsystem, not a vocabulary type.
- TartanLlama/optional — the sister single-header library providing `tl::optional` with the same functional extensions. Use when you need "value or nothing" rather than "value or error".
- The C++23 standard `std::expected` — use when your toolchain supports it; there is no reason to prefer a backport on a conforming compiler.
- bloomberg/bde `bdlb::NullableValue` and status-code patterns — use when you are already in an established house framework with its own error conventions.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1 | 2017-10 | Initial single-header release tracking p0323r3[^1]. |
| v0.2 | 2017-12 | Early fixes; broadened compiler coverage. |
| v0.3 | 2018-10 | Continued conformance and trait-propagation work. |
| v1.0.0 | 2019-06-25 | First stable tag. |
| v1.1.0 | 2023-03-15 | Maintenance release after a long gap. |
| v1.2.0 | 2025-07-12 | Resumed maintenance; fixes after ~2-year idle period[^2]. |
| v1.3.1 | 2025-09-01 | Housekeeping bump (released same day as v1.3.0). |

## References

[^1]: Vicente J. Botet Escriba & JF Bastien, "A proposal to add a utility class to represent expected monad" (p0323r3), WG21, 2017. http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0323r3.pdf
[^2]: TartanLlama/expected releases — tag dates via GitHub Releases API (v1.1.0 2023-03-15, v1.2.0 2025-07-12, v1.3.1 2025-09-01). https://github.com/TartanLlama/expected/releases

## Tags

cpp, c++11, header-only, error-handling, expected, monadic, functional, result-type, std-expected, vocabulary-type, no-exceptions
