# ericniebler/range-v3

> The C++ range library that became the prototype for C++20's `std::ranges` — composable, lazy views layered on top of iterators.

[GitHub repo](https://github.com/ericniebler/range-v3) ·
[Documentation](https://ericniebler.github.io/range-v3/) ·
[License: BSL-1.0 (Boost)](https://github.com/ericniebler/range-v3/blob/master/LICENSE.txt)

## Overview

range-v3 is a header-only C++14/17/20 library by Eric Niebler that extends the STL's iterators and algorithms with a notion of *ranges* — objects with a `begin()`/`end()` pair — and makes those ranges *composable*. It was the reference implementation behind the formal standardization effort: N4128 (2014) laid out the rationale, the design became the Ranges TS, and finally P0896R4 "The One Ranges Proposal" was merged into the C++20 working draft in November 2018[^1]. Much of what shipped as `<ranges>` in C++20 is a subset of this library.

The design choice that separates range-v3 from earlier attempts is that ranges are an abstraction *on top of* iterators rather than a replacement for them[^2]. The library is organized around three pillars: **Algorithms** (the familiar STL algorithms, plus range-taking overloads and projections), **Views** (lazy, non-owning adaptations composed with the pipe operator, e.g. `rng | views::filter(f) | views::transform(g)`), and **Actions** (eager, in-place mutations of an owning container such as `v |= actions::sort | actions::unique`). Views compute nothing until iterated; actions run immediately and return the mutated container for further chaining.

The library's defining tension is expressiveness versus compile-time and diagnostic cost. Because it predates language-level concepts, range-v3 emulates them with a macro layer, and pushes template instantiation hard. The payoff is Python-comprehension-style pipelines in C++; the price is long build times, deep instantiation stacks, and error messages that were notoriously large before compiler concepts support landed. It remains far broader than the standardized subset — actions, for instance, were never standardized at all.

## Getting Started

Header-only. Fetch it via a package manager (vcpkg `range-v3`, Conan `range-v3/[*]`) or CMake `FetchContent`, then link the `range-v3::range-v3` interface target.

```cmake
include(FetchContent)
FetchContent_Declare(range-v3
  GIT_REPOSITORY https://github.com/ericniebler/range-v3
  GIT_TAG 0.12.0)
FetchContent_MakeAvailable(range-v3)
target_link_libraries(myapp PRIVATE range-v3::range-v3)
```

```cpp
#include <range/v3/all.hpp>
#include <iostream>
#include <vector>

int main() {
    std::vector<int> v{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Views: lazy, left-to-right, evaluated only in the loop.
    auto squares_of_evens = v
        | ranges::views::filter([](int i){ return i % 2 == 0; })
        | ranges::views::transform([](int i){ return i * i; });

    for (int i : squares_of_evens)
        std::cout << i << ' ';   // 4 16 36 64 100
    std::cout << '\n';

    // Actions: eager, mutate the container in place.
    v |= ranges::actions::sort | ranges::actions::unique;
    return 0;
}
```

Compile with `-std=c++17` (or `c++14`/`c++20`). On MSVC the library requires `/permissive-` and `/std:c++17` or later.

## Architecture / How It Works

**Concept emulation.** range-v3 carries its own concepts layer (the `CPP_*` / `concepts::` machinery) that expands either to real C++20 concepts on new compilers or to SFINAE-based emulation on older ones[^3]. This is why it compiles on GCC 6.5 and Clang 5 but also why templated call sites are heavier than hand-written iterator code.

**Views are non-owning and lazy.** A view stores its source range (by reference or by a cheap handle) plus the adaptor's function objects, and computes elements on demand through a custom iterator. Composition builds a nested iterator type at compile time — `filter | transform` produces one fused iterator, not two passes. Because a view does not own its elements, feeding it a temporary container is a dangling-reference bug the type system only partially catches.

**Actions own and mutate.** An action requires an lvalue container (or takes ownership of an rvalue), applies an algorithm eagerly, and returns the container. `sort`, `unique`, `shuffle`, `push_back`, `slice`, etc. Actions have no counterpart in standard `<ranges>`.

**Algorithms and projections.** Every algorithm has an iterator-pair overload and a range overload, and accepts a *projection* — a callable applied to each element before the predicate/comparator, so you can `sort(people, less{}, &Person::age)` without a custom comparator.

**The `ranges::cpp20` namespace.** Components mirroring what was standardized live under `ranges::cpp20`. The maintainer commits to keeping that subset stable; everything outside it may change without regard to backward compatibility[^4]. If you intend to migrate to `std::ranges` later, staying inside `cpp20::` keeps the surface close.

## Production Notes

**Compile time is the dominant cost.** Heavy `#include <range/v3/all.hpp>` pulls in most of the library; prefer the granular headers (`range/v3/view/filter.hpp`, `range/v3/action/sort.hpp`) to keep translation units lean. Deep view pipelines instantiate deep template stacks, and this compounds across a codebase.

**Debug builds are slow at runtime.** View pipelines depend on the optimizer to inline the layered iterator calls away. Without `-O2`/`-O3` (or an MSVC release config), a `filter | transform | take` chain can be dramatically slower than an equivalent hand loop because every element access crosses several un-inlined iterator adaptor frames. Benchmark in release, not debug.

**Dangling and lifetime footguns.** Views don't own. `auto v = get_vector() | views::filter(pred);` binds to a temporary that dies at the end of the full expression; iterating `v` afterward is undefined behavior. Materialize with `ranges::to<std::vector>(...)` when you need to outlive the source. Single-pass (input) views also cannot be traversed twice.

**MSVC has historically been the weak platform.** The library's strict conformance means it needs `/permissive-` and a recent Visual Studio; older MSVC releases mis-compiled parts of it. GCC and Clang are the better-supported toolchains.

**It is not a drop-in for `std::ranges`.** The standard adopted a subset with renamed or subtly different semantics, added its own views over C++20/23/26, and omitted actions entirely. Code written against range-v3's full surface will not port mechanically to `std::ranges`; only the `cpp20::` subset is close. Conversely, if your compiler already ships a complete `<ranges>`, much of range-v3's day-to-day value is already in the standard library.

**Stability posture.** The README states plainly that outside `ranges::cpp20`, no promise of support or long-term stability is made and the code will evolve without regard to backward compatibility. Pin a tagged release rather than tracking `master`.

## When to Use / When Not

**Use when:**
- You are on C++14/17 (or a toolchain with incomplete `<ranges>`) and want composable lazy views today.
- You need capabilities beyond standard `<ranges>`: actions, the wider set of views, or projection-taking algorithms across the whole STL.
- You want a battle-tested implementation whose design directly informed the standard.

**Avoid when:**
- You target C++20/23 with a modern stdlib and only need the standardized subset — use `<ranges>` and drop the dependency.
- Compile time or debug-build performance is a hard constraint on your project.
- Your team is uncomfortable with template-heavy error messages and long instantiation chains.
- You need MSVC support on an older Visual Studio, or a permissive/non-conforming compiler mode.

## Alternatives

- The C++ standard `<ranges>` — use instead when your compiler ships it and you only need the standardized views/algorithms; no third-party dependency.
- tcbrindle/flux — use instead when you want a C++20 sequence library with a cursor model aimed at better codegen and bounds safety, and don't need range-v3's breadth.
- tcbrindle/NanoRange — use instead when you want a lighter, single-header C++17 implementation tracking the Ranges TS with faster compiles.
- boostorg/range — use instead when you're already deep in Boost and want the older Boost.Range adaptors without adding a new dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| N4128 | 2014 | Rationale paper; the design's public starting point[^1]. |
| Ranges TS | 2015–2017 | "C++ Extensions for Ranges" technical specification. |
| P0896R4 | 2018-11 | "The One Ranges Proposal" merged into the C++20 draft[^1]. |
| 0.10.0 | 2020 | Pre-`<ranges>` era release. |
| 0.11.0 | 2020 | Widely packaged stable tag. |
| 0.12.0 | 2022 | Most recent tagged release as of 2026. |

## References

[^1]: range-v3 README and P0896R4 "The One Ranges Proposal," merged into the C++20 working draft in November 2018. https://wg21.link/p0896r4
[^2]: Eric Niebler, project README — ranges as an abstraction layer on top of iterators, built on Views, Actions, and Algorithms. https://github.com/ericniebler/range-v3
[^3]: N4381 "Suggested Design for Customization Points" and Niebler, "Concept checking in C++11." http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/n4381.html
[^4]: range-v3 README, "Development Status" — the `ranges::cpp20` namespace is the stable subset; the rest evolves without backward-compatibility guarantees. https://github.com/ericniebler/range-v3#readme

## Tags

cpp, c-plus-plus, ranges, iterators, header-only, functional, lazy-evaluation, algorithms, stl, cpp20, metaprogramming, library
