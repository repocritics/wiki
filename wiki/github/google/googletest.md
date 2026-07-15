# google/googletest

> Google's C++ xUnit test framework, bundled with the GoogleMock mocking library.

[GitHub repo](https://github.com/google/googletest) ·
[Official website](https://google.github.io/googletest/) ·
[License: BSD-3-Clause](https://github.com/google/googletest/blob/main/LICENSE)

## Overview

GoogleTest is a C++ unit-testing framework built on the xUnit model (test suites, fixtures, setup/teardown, assertions)[^1]. Open-sourced by Google in 2008, it is one of the most widely deployed C++ test frameworks: Chromium, LLVM, Protocol Buffers, and OpenCV all use it[^2]. This repository is the result of merging the formerly separate **GoogleTest** and **GoogleMock** projects, which were released independently until they were folded into a single tree because they were almost always used together[^1].

The framework's defining characteristic is that it is a *matched pair*. GoogleTest provides the runner and assertions (`EXPECT_EQ`, `ASSERT_TRUE`, matchers, death tests, value- and type-parameterized tests); GoogleMock provides mock-object generation (`MOCK_METHOD`, `EXPECT_CALL`, cardinalities, matchers, actions). The two share a matcher vocabulary, so `EXPECT_THAT(value, ElementsAre(...))` and `EXPECT_CALL(mock, F(Gt(3)))` read the same way. Teams that adopt GoogleTest generally inherit GoogleMock whether they wanted it or not.

The central tradeoff is macro-heaviness versus completeness. GoogleTest is a large, macro-driven framework — it does far more than lighter competitors (Catch2, doctest), but at the cost of longer compile times, heavier headers, and a build integration story that has historically been the main friction point. It is the safe institutional default for C++ testing, not the minimalist one.

## Getting Started

With CMake via `FetchContent` (the maintainers' recommended path)[^3]:

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.17.0.zip
)
FetchContent_MakeAvailable(googletest)

enable_testing()
add_executable(my_test my_test.cc)
target_link_libraries(my_test GTest::gtest_main GTest::gmock)
include(GoogleTest)
gtest_discover_tests(my_test)
```

```cpp
#include <gtest/gtest.h>

int Add(int a, int b) { return a + b; }

TEST(AddTest, HandlesPositives) {
  EXPECT_EQ(Add(2, 3), 5);   // non-fatal: keeps going on failure
  ASSERT_NE(Add(2, 3), 0);   // fatal: aborts this test on failure
}
```

Linking `GTest::gtest_main` provides a `main()`; drop it and write your own if you need custom setup. Bazel is a first-class alternative to CMake and is how Google builds it internally.

## Architecture / How It Works

A `TEST(Suite, Name)` macro expands into a subclass of `::testing::Test` plus a static registration object whose constructor adds the test to a global registry before `main()` runs. `RUN_ALL_TESTS()` walks that registry. This "automatic discovery via static initializers" is why no manual test-list registration is needed — and why tests in a static library can silently vanish if the linker drops the translation unit as unreferenced (see Production Notes).

Assertions come in two severities that are structurally different: `EXPECT_*` records a failure and continues; `ASSERT_*` records a failure and does a `return` from the current function. Because `ASSERT_*` is a hidden `return`, it cannot be used in a function with a non-void return type, and it will not run destructors of anything you expected to clean up later in the test body. This surprises people migrating from frameworks that use exceptions for failures.

GoogleMock generates mocks at compile time. `MOCK_METHOD(ret, name, (args), (specs))` expands to an override plus the plumbing that lets `EXPECT_CALL` set up expectations. Expectations are verified in the mock's destructor by default, so an unmet `EXPECT_CALL` reports failure when the mock goes out of scope — meaning mock lifetime, not test-body position, determines when failures surface. Uninteresting-call warnings, strict/nice/naggy mocks, and call ordering (`InSequence`) are all part of this layer.

Matchers (`Gt`, `HasSubstr`, `ElementsAre`, `Pointee`, `Field`) are shared between the two halves via `EXPECT_THAT`. Death tests fork the process (on POSIX) or spawn a subprocess to assert that code aborts or exits with a given status; they interact badly with threads because forking a multi-threaded process is undefined, which the framework warns about at runtime.

## Production Notes

**Tests disappearing from static libraries.** The most common footgun: if your tests live in a `.a`/`.lib` and nothing references their symbols, the linker discards the object files and the tests never register. Fixes are linker-specific (`--whole-archive` on GNU ld, `/WHOLEARCHIVE` on MSVC, `alwayslink` in Bazel) — link test objects directly into the test binary or force whole-archive inclusion.

**Compile time and header weight.** GoogleTest/GoogleMock headers are large and template-heavy. In big test suites the framework itself, not your code, can dominate compile time. This is the recurring reason teams evaluate doctest or Catch2 for fast inner-loop testing.

**C++ standard floor keeps rising.** Older code pinned to a specific GoogleTest tag because upgrades moved the minimum standard. Recent releases require newer standards — the 1.17.x line requires at least C++17[^4]. Upgrading GoogleTest can force a compiler/standard bump across the whole test target.

**Abseil dependency incoming.** The maintainers have announced plans to take a dependency on Abseil[^1]. Historically GoogleTest was self-contained; a transitive Abseil dependency changes the vendoring calculus for projects that pull GoogleTest into constrained or offline builds. Watch this if you vendor sources by hand.

**`ASSERT_*` control flow.** Because fatal assertions `return`, using them inside helper subroutines does not abort the *test* — only the helper. Assertions that need to stop the whole test must sit in the test body, or the helper must signal upward. This is a frequent source of tests that "pass" after a fatal assertion in a helper.

**Death tests and threads.** Death tests fork; forking a multithreaded process is unsafe. Under thread sanitizers or in threaded fixtures they are flaky or hang. GoogleTest emits a warning but cannot prevent it.

## When to Use / When Not

**Use when:**
- You need mocking; GoogleMock is the most complete mocking library in the C++ ecosystem.
- You want the institutional default that every C++ engineer recognizes and CI/IDE tooling already supports.
- You need death tests, type-parameterized tests, or a rich matcher library out of the box.
- You are already on CMake or Bazel and can absorb the build integration.

**Avoid when:**
- Compile time is a priority and you want a header-only, near-zero-setup framework (doctest, Catch2).
- You want a minimal, macro-light, modern-C++ style test framework (boost-ext/ut).
- You are on a constrained/embedded target that cannot carry GoogleTest's footprint (or a future Abseil dependency).
- You do not need mocking and resent pulling in GoogleMock's surface area.

## Alternatives

- catchorg/Catch2 — single-header (v2) / modular (v3) framework; use when you want lighter setup and BDD-style sections and don't need GoogleMock.
- doctest/doctest — the fastest to compile; use when test compile time dominates or you want tests co-located in production source behind a flag.
- boost-ext/ut — macro-free, C++20, minimal footprint; use when you want modern syntax without the macro machinery.
- cpputest/cpputest — C-friendly with built-in memory-leak detection; use for embedded and mixed C/C++ codebases.
- rollbear/trompeloeil — a standalone mocking library; use when you keep GoogleTest (or another runner) but want mocking with different ergonomics.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2008 | Google Test open-sourced as a standalone C++ framework[^2]. |
| — | ~2016 | GoogleTest and GoogleMock merged into one repository[^1]. |
| 1.13.0 | 2023 | Raised the minimum language standard to C++14. |
| 1.14.0 | 2023 | C++14 baseline; CMake/Bazel-focused build guidance. |
| 1.15.x | 2024 | Continued C++14 support line. |
| 1.17.0 | 2025 | Requires at least C++17[^4]. |

## References

[^1]: GoogleTest README — merger of GoogleTest and GoogleMock, announcements, planned Abseil dependency. https://github.com/google/googletest/blob/main/README.md
[^2]: GoogleTest README, "Who Is Using GoogleTest?" — Chromium, LLVM, Protocol Buffers, OpenCV. https://github.com/google/googletest/blob/main/README.md
[^3]: GoogleTest documentation, "Quickstart: Building with CMake". https://google.github.io/googletest/quickstart-cmake.html
[^4]: GoogleTest 1.17.0 release; the 1.17.x branch requires at least C++17 per the README and Google's Foundational C++ Support Policy. https://github.com/google/googletest/releases/tag/v1.17.0

## Tags

cpp, testing, unit-testing, mocking, xunit, test-framework, gmock, cmake, bazel, google
