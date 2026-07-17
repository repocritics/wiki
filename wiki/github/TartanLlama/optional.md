# TartanLlama/optional

> A single-header C++11 `std::optional` with monadic extensions and, uniquely, support for optional references.

[GitHub repo](https://github.com/TartanLlama/optional) ·
[Official website](https://tl.tartanllama.xyz) ·
[License: CC0-1.0](https://github.com/TartanLlama/optional/blob/master/COPYING)

## Overview

`tl::optional` is a header-only reimplementation of `std::optional` by Sy Brand (TartanLlama), first published in 2017[^1]. It exists for two reasons that were true at the time and are now only half true. First, it backports `std::optional` (a C++17 type) to C++11 and C++14 compilers. Second, and more durably, it adds functional-style combinators — `map`, `and_then`, `or_else`, and relatives — that let you chain fallible computations without a ladder of `if (!opt) return std::nullopt;` checks.

The library doubles as the reference implementation of WG21 paper P0798, "Monadic operations for std::optional"[^2]. That proposal succeeded: C++23 added `transform` (the standard's name for `map`), `and_then`, and `or_else` directly to `std::optional`[^3]. Since then, `tl::optional`'s monadic value proposition applies mainly to codebases stuck on C++11/14/17 toolchains where the standard type lacks those methods.

The feature that C++23 did *not* absorb — and the reason the library still has a reason to exist on modern compilers — is `tl::optional<T&>`. The standard `std::optional` forbids reference template arguments; `tl::optional` supports them with rebind-on-assignment semantics, giving you a nullable, reseatable non-owning handle. That is a genuinely different tool from `std::optional`, `T*`, or `std::reference_wrapper`, and it is the main thing users reach for the library for in 2026.

## Getting Started

There is nothing to build or link. Copy `include/tl/optional.hpp` into your tree, or add the repo as a submodule / CMake subdirectory and link the interface target `tl::optional`.

```cpp
#include <tl/optional.hpp>
#include <string>

tl::optional<int> parse_int(const std::string&);  // returns nullopt on failure

// Chain fallible steps; the whole expression is nullopt if any step fails.
tl::optional<std::size_t> len =
    tl::make_optional(std::string{"1234"})
        .and_then(parse_int)                 // optional<int>
        .map([](int n){ return n * 2; })     // optional<int>
        .map([](int n){ return std::to_string(n).size(); });  // optional<size_t>

// Optional reference: a nullable, reseatable handle.
int i = 42;
tl::optional<int&> r = i;
*r = 7;            // writes through to i
int j = 8;
r = j;             // rebinds — r now refers to j, does NOT assign 8 into i
```

`and_then` is for step functions that themselves return an optional; `map` is for functions that return a plain value and wraps the result. `or_else` runs a fallback when empty. `map_or` / `map_or_else` collapse to a non-optional with a default.

## Architecture / How It Works

The whole library is one header. Internally it is the usual optional layout: an aligned storage buffer plus a `bool` engaged flag, wrapped in a chain of base classes that are conditionally `constexpr`, conditionally trivially-copyable, and conditionally trivially-destructible depending on the stored type's own traits. This SFINAE/trait plumbing is what lets `tl::optional<T>` be a literal type when `T` is trivial, matching `std::optional`'s guarantees rather than pessimizing everything into a non-trivial type.

The monadic methods are thin. `map(f)` inspects the return type of `f`; if `f` returns `void` it yields `tl::optional<monostate>`, otherwise `tl::optional<decltype(f(value))>`. `and_then(f)` requires `f` to already return a `tl::optional` and simply forwards it or short-circuits. All of them are overloaded across the `&`, `const&`, `&&`, and `const&&` value categories, which is most of the header's line count.

The reference specialization `tl::optional<T&>` is a separate, much simpler class: it holds a `T*` and treats `nullptr` as the empty state. This is why assignment rebinds instead of assigning through — under the hood you are reseating a pointer, not calling `T::operator=`. That design choice is deliberate and documented, but it is the single most common source of surprise, because `std::optional<T>` (value) *does* assign through. Same syntax, opposite semantics.

There are no external dependencies. Compatibility shims (`in_place_t`, `monostate`, `bad_optional_access`) are defined locally so the header works standalone on C++11.

## Production Notes

- **Semantic drift versus `std::optional<T&>` proposals.** `tl::optional<T&>` predates and differs from the reference-optional behavior that later WG21 discussions leaned toward (some favored assign-through). If a future standard ships optional references with different assignment semantics, code that relies on `tl::optional`'s rebind behavior will not port cleanly. Treat the rebind semantics as a library-specific contract, not a standard one.
- **Reaching for this on C++23 is often redundant.** If your toolchain has C++23 `std::optional`, you already have `transform`/`and_then`/`or_else`. Pulling in `tl::optional` only for the monadic ops adds a second optional type to your codebase and the friction of converting between the two at API boundaries. Adopt it selectively — for the reference support, or for C++11/14/17 targets — not as a blanket replacement.
- **Two optional types is a real cost.** Mixing `tl::optional` and `std::optional` in one codebase means functions that take one won't accept the other without conversion, and overload sets get noisy. Pick one at each interface boundary.
- **Method name mismatch with the standard.** The library calls it `map`; C++23 calls it `transform`. Code migrated off `tl::optional` toward the standard type needs a rename, and mental-model whiplash for reviewers who know only one name.
- **Maintenance cadence is low.** The library is stable and largely feature-complete; the last commit to `master` was mid-2024[^4]. That is appropriate for a small header whose scope is fixed, but do not expect active feature development — it is in "done" mode, not "growing" mode.
- **Compiler floor.** Advertised support reaches back to clang 3.5 and g++ 4.8, plus MSVC 2015/2017[^1]. On very old MSVC the trait-heavy code paths were historically the flakiest; modern compilers are fine.

## When to Use / When Not

**Use when:**
- You need a nullable, reseatable reference (`tl::optional<T&>`) — the standard type cannot express this.
- You are targeting C++11/14/17 and want monadic `map`/`and_then`/`or_else` today.
- You want a zero-dependency, public-domain single header you can vendor without license friction.

**Avoid when:**
- You are on C++23: prefer standard `std::optional` with `transform`/`and_then`/`or_else`.
- You want richer error information on the empty path — an optional discards the reason; use an expected/result type instead.
- You want to avoid maintaining two optional types across an API surface.

## Alternatives

- TartanLlama/expected — same author; use when the empty case needs to carry an error value, not just "nothing."
- martinmoene/optional-lite — use when you want an optional backport with explicit C++ standard-version fallbacks and wide compiler matrixing.
- The standard `std::optional` — use when you are on C++23 and don't need reference support; the monadic methods are built in.
- akrzemi1/Optional — use as the historical reference optional predating standardization; mostly of archival interest now.
- Boost.Optional (boostorg/optional) — use when you already depend on Boost and want its long-established optional, including its own reference support.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-09-29 | First public release; single-header optional with monadic extensions[^1]. |
| P0798R0 | 2017 | Serves as reference implementation for the monadic-operations proposal[^2]. |
| — | 2024-06-10 | Last commit to `master`; library in stable/maintenance mode[^4]. |
| C++23 | 2023 | Standard `std::optional` gains `transform`/`and_then`/`or_else`, absorbing the proposal's core[^3]. |

## References

[^1]: TartanLlama/optional README and repository. https://github.com/TartanLlama/optional
[^2]: WG21 P0798, "Monadic operations for std::optional." https://wg21.tartanllama.xyz/monadic-optional
[^3]: cppreference, `std::optional` monadic operations (`transform`, `and_then`, `or_else`), C++23. https://en.cppreference.com/w/cpp/utility/optional
[^4]: GitHub repository metadata, last push 2024-06-10 (fetched 2026-07). https://github.com/TartanLlama/optional

## Tags

cpp, header-only, optional, monadic, functional, std-optional, backport, reference-semantics, error-handling, public-domain
