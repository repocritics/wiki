# catchorg/Catch2

> A C++ test framework whose defining trick is that `REQUIRE(a == b)` prints both `a` and `b` on failure — no `EXPECT_EQ` family required.

[GitHub repo](https://github.com/catchorg/Catch2) ·
[Documentation](https://github.com/catchorg/Catch2/blob/devel/docs/Readme.md) ·
[License: BSL-1.0](https://github.com/catchorg/Catch2/blob/devel/LICENSE.txt)

## Overview

Catch2 is a unit-testing framework for C++, originally written by Phil Nash and now maintained primarily by Martin Hořeňovský[^1]. The name is a backronym: "C++ Automated Test Cases in Headers." Its signature feature is **expression decomposition**: you write ordinary boolean expressions inside `REQUIRE(...)` / `CHECK(...)`, and template + operator-overload machinery captures the left- and right-hand operand values so the failure message shows what the values actually were — without the `ASSERT_EQ` / `ASSERT_LT` / `ASSERT_NE` zoo that GoogleTest requires.

The framework layers three testing styles on one engine: plain unit tests (`TEST_CASE` + `SECTION`), BDD macros (`SCENARIO` / `GIVEN` / `WHEN` / `THEN`, which are thin aliases over the same primitives), and a micro-benchmarking mode derived from the Nonius library[^2]. It ships with no third-party dependencies, which is a large part of why it spread: dropping it into a project historically meant copying one header.

The central tension is compile time versus ergonomics. Expression decomposition and heavy header-only templating made Catch2 pleasant to write against but slow to compile at scale — the reason v3 abandoned the single-header model (see Architecture). It targets C++14 and later as of v3; the v2.x branch remains the C++11 option, and the original Catch1.x branch covers C++03[^3].

## Getting Started

The recommended route for v3 is CMake with `FetchContent`, or a system package (vcpkg, Conan, apt):

```cmake
include(FetchContent)
FetchContent_Declare(
  Catch2
  GIT_REPOSITORY https://github.com/catchorg/Catch2.git
  GIT_TAG        v3.5.0   # pin an actual release tag
)
FetchContent_MakeAvailable(Catch2)

add_executable(tests test.cpp)
target_link_libraries(tests PRIVATE Catch2::Catch2WithMain)
```

```cpp
// test.cpp
#include <catch2/catch_test_macros.hpp>

int factorial(int n) { return n <= 1 ? 1 : factorial(n - 1) * n; }

TEST_CASE("factorials are computed", "[math]") {
    REQUIRE(factorial(1) == 1);
    REQUIRE(factorial(3) == 6);

    SECTION("larger inputs") {
        REQUIRE(factorial(10) == 3'628'800);
    }
}
```

Linking `Catch2::Catch2WithMain` provides `main()`; link `Catch2::Catch2` instead when you supply your own entry point.

## Architecture / How It Works

**Expression decomposition.** `REQUIRE(a == b)` expands to a macro that funnels the expression into an `Expr` capturing type. Operator overloads on the decomposer bind the LHS, then the comparison operator captures the RHS, so both operands are stringified (via `Catch::StringMaker`) only if the assertion fails. The cost: every assertion instantiates templates, and the operator-precedence trick only works one level deep — `REQUIRE(a == b == c)` does not compile, and complex expressions must be wrapped or split.

**Sections are a re-execution tree.** A `TEST_CASE` containing `SECTION`s is run once per leaf section. On each run the framework walks the section tree, executing exactly one leaf path and the setup/teardown code lexically surrounding it. This gives local, nestable fixtures without fixture classes, but it means shared setup code above a `SECTION` runs again for every leaf — an O(leaves) cost that surprises people who put expensive work at the top of a test case.

**v3 is a compiled library, not a header.** Through v2, Catch2 shipped as `catch.hpp`, a single amalgamated header. v3 (first release 3.0.1, 2022) split it into many focused headers plus a separately compiled static library[^4]. You now include only `catch2/catch_test_macros.hpp`, `catch2/matchers/...`, `catch2/benchmark/...` as needed, and link the built library. This was the headline change and the main migration friction from v2.

**Registration is static-init driven.** `TEST_CASE` defines an anonymous object whose constructor registers the test with a global registry before `main()`. This is why tests need no manual listing but also why they cannot be conditionally registered at compile time except through tags and runtime filtering.

**Peripheral systems:** `GENERATE(...)` for data-driven cases, a matcher library (`Catch::Matchers`) for expressive assertions on containers/strings/floats, and pluggable reporters (console, JUnit XML, SonarQube, TAP, XML) selectable with `-r`. CMake integration ships as `Catch.cmake` with `catch_discover_tests()`, which registers each `TEST_CASE` as an individual CTest test by parsing the binary's test list at build time[^5].

## Production Notes

**Compile time is the recurring complaint.** Even in v3, including the test macros pulls in nontrivial templates, and expression decomposition instantiates per assertion. Large suites (thousands of `TEST_CASE`s) compile noticeably slower than the same suite in doctest. Mitigations: keep test translation units small, avoid putting Catch includes in headers, and precompile the Catch header. Teams that live or die by iteration speed frequently choose doctest specifically for this reason.

**`catch_discover_tests` vs binary size and build time.** Registering every test with CTest requires running the test binary at build time to enumerate tests. On very large suites this enumeration and the resulting CTest fan-out add measurable configure/build overhead; the coarser `add_test(NAME all COMMAND tests)` avoids it at the cost of per-test CTest granularity.

**Section semantics are a footgun.** Because the case body re-runs per leaf section, code with side effects or non-idempotent setup above a `SECTION` executes repeatedly. Static/global state mutated in one section is visible in the next unless reset. This is by design but routinely trips newcomers migrating fixture-heavy GoogleTest suites.

**v2 → v3 migration is not free.** Include paths change (`catch.hpp` → granular headers), you must link a real library and adjust your build system, some macros moved, and `CATCH_CONFIG_MAIN` gives way to linking `Catch2WithMain`[^3]. Projects that vendored the single header now take a dependency on a compiled artifact. Many codebases stayed on v2.x for years for this reason; v2.x still receives occasional fixes but not new features.

**No built-in mocking.** Catch2 asserts; it does not mock. Pair it with a separate mocking library (FakeIt, Trompeloeil, GoogleMock used standalone) or hand-rolled test doubles. Benchmarks are opt-in — cases tagged `[!benchmark]` do not run unless requested.

## When to Use / When Not

**Use when:**
- You want natural-language test names and assertions that read as plain C++ booleans with automatic value reporting.
- You value zero external dependencies and easy CMake/vcpkg/Conan integration.
- You want unit tests, BDD-style specs, and micro-benchmarks from one framework.
- You are on C++14+ and can absorb a compiled test-library dependency.

**Avoid when:**
- Compile time dominates your feedback loop and the suite is large — doctest or ut compile substantially faster.
- You need integrated mocking as a first-class feature — GoogleTest + GoogleMock is more cohesive.
- You are stuck on C++03/C++11 and cannot use newer branches cleanly.
- You want a single-header drop-in in v3 — that model was retired; use v2.x or doctest instead.

## Alternatives

- google/googletest — heavier and macro-family-based assertions, but integrated GoogleMock and the widest industry adoption; use it when you need mocking and death tests in one package.
- doctest/doctest — deliberately Catch-compatible API with far faster compile times and a true single header; use it when build speed or in-production-code tests matter.
- boost-ext/ut — C++20, macro-free, extremely fast to compile; use it when you are on a modern toolchain and want minimal preprocessor magic.
- cpputest/cpputest — C/C++ with built-in memory-leak detection and mocking; use it in embedded/legacy C contexts.
- boostorg/test (Boost.Test) — mature and full-featured; use it when you already depend on Boost.

## History

| Version | Date | Notes |
|---------|------|-------|
| Catch1.x | 2010–2017 | Original single-header framework (Phil Nash); C++03 branch retained. |
| 2.0 | 2017-10 | Single-header, C++11 baseline; expression decomposition matured[^3]. |
| 3.0.1 | 2022-06 | First v3 release: compiled library, granular headers, C++14 baseline[^4]. |
| 3.x | 2022– | Ongoing v3 line on the `devel` branch; matchers, generators, reporters expanded. |

## References

[^1]: Catch2 repository and maintainer history, catchorg/Catch2. https://github.com/catchorg/Catch2
[^2]: Catch2 benchmarking docs (derived from the Nonius micro-benchmarking library). https://github.com/catchorg/Catch2/blob/devel/docs/benchmarks.md
[^3]: Catch2 v2-to-v3 migration guide. https://github.com/catchorg/Catch2/blob/devel/docs/migrate-v2-to-v3.md
[^4]: Catch2 release notes / v3.0.1 release. https://github.com/catchorg/Catch2/releases
[^5]: Catch2 CMake integration (`catch_discover_tests`). https://github.com/catchorg/Catch2/blob/devel/docs/cmake-integration.md

## Tags

cpp, testing, unit-testing, test-framework, tdd, bdd, header-only, benchmarking, cmake, c-plus-plus
