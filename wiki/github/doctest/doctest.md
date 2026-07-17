# doctest/doctest

> Single-header C++ testing framework built for compile-time speed, designed so tests can live inside the production code they cover.

[GitHub repo](https://github.com/doctest/doctest) ·
[Documentation](https://github.com/doctest/doctest/blob/master/doc/markdown/readme.md) ·
[License: MIT](https://github.com/doctest/doctest/blob/master/LICENSE.txt)

## Overview

doctest is a header-only unit-testing framework for C++11 and later, first released in 2016 by Viktor Kirilov[^1]. It is modeled after Catch — the assertion macros, `SUBCASE`/`SECTION` model, and expression-decomposition style are deliberately familiar — but rewritten with one overriding constraint: the header must be cheap to include and cheap to instantiate. The selling point is that adding `#include <doctest/doctest.h>` and writing thousands of assertions costs far less compile time than the feature-comparable alternatives[^2], which is what makes the framework's headline idea viable: writing tests at the bottom of the same `.cpp` (or `.h`) file as the code they exercise.

That "tests next to production code" model is the framework's defining bet. Because the header drags in almost no standard-library headers and everything lives in the `doctest::` namespace, you can leave test code compiled into a normal build during development and then strip every trace of it from release binaries by defining `DOCTEST_CONFIG_DISABLE`[^3]. In practice most teams still use doctest the conventional way — a separate test target — and treat the compile-speed win and the clean, warning-free header as the reason to pick it over Catch2 or GoogleTest.

The tension worth understanding up front: doctest optimizes hard for the edit/compile/test inner loop and for being unintrusive, not for batteries-included breadth. It has no built-in mocking, no test-level parallel execution, and a deliberately small runtime. Teams that want a mocking framework, death tests, or sharded parallel runs bolt those on separately or choose GoogleTest instead.

## Getting Started

doctest is a single header. Vendor `doctest/doctest.h`, or install via a package manager (vcpkg, Conan, apt `doctest-dev`, Homebrew `doctest`), or `find_package(doctest)` after a CMake install.

```cpp
// test_main.cpp — exactly one translation unit defines the implementation + main()
#define DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN
#include <doctest/doctest.h>

int factorial(int n) { return n <= 1 ? 1 : n * factorial(n - 1); }

TEST_CASE("factorial computes small values") {
    CHECK(factorial(1) == 1);
    CHECK(factorial(3) == 6);

    SUBCASE("edge cases") {
        CHECK(factorial(0) == 1);   // whole TEST_CASE re-runs once per SUBCASE leaf
    }
}
```

```bash
g++ -std=c++17 test_main.cpp -o tests && ./tests
./tests --test-case="factorial*" --success   # filter + show passing asserts
```

## Architecture / How It Works

Test cases self-register at static-initialization time: `TEST_CASE(...)` expands to a uniquely-named static object whose constructor pushes the test into a global registry before `main()` runs. The `--test-case`, `--test-suite`, and other filters then select from that registry at runtime.

Exactly one translation unit must supply the implementation, via `DOCTEST_CONFIG_IMPLEMENT` (you write `main`) or `DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN` (doctest writes it). Every other TU includes the header in its default, declaration-only mode. This split is what keeps per-file compile cost low — the heavy machinery is instantiated once.

Subcases are doctest's answer to fixtures. Rather than a setup/teardown class, a `TEST_CASE` re-executes from the top once for each leaf `SUBCASE`, so code before a subcase acts as shared setup and code in a subcase branch runs in isolation from sibling branches. Traditional class fixtures (`TEST_CASE_FIXTURE`) and typed/templated test cases also exist. Assertions come in three severities — `CHECK` (log and continue), `REQUIRE` (abort the test case via exception), and `WARN` — each with `_FALSE`, `_THROWS`, `_THROWS_AS`, `_NOTHROW`, and `_MESSAGE` variants. Expression decomposition means `CHECK(a == b)` prints both operand values on failure with no special macro, provided the types are stringifiable.

The runtime is intentionally small and single-threaded at the test level: doctest runs test cases sequentially. What it does guarantee is that assertion macros are thread-safe *within* a single test case, so you can fire asserts from threads you spawn inside one test[^4]. Output is pluggable through the reporter interface (console and JUnit XML ship in-box; custom reporters register like tests do), which is the integration point for CI dashboards and CTest.

## Production Notes

- **Tests in a static library silently vanish.** Because registration relies on static-initializer side effects, a linker that sees no referenced symbol in a test-only object file will drop the whole translation unit — and your tests with it. This is the single most common doctest surprise. Compile test files into the test executable directly, or force inclusion (`-Wl,--whole-archive`, MSVC `/WHOLEARCHIVE`, or an explicit reference). This is a property of C++ linking, not a doctest bug, but it bites everyone once.
- **Exactly one implementation definition.** Defining `DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN` in two TUs gives duplicate-symbol / duplicate-`main` link errors; forgetting it entirely gives undefined references. Keep it in one dedicated file.
- **`DOCTEST_CONFIG_DISABLE` strips test bodies, not your mistakes.** With it defined, `TEST_CASE` blocks are removed from the binary, but any non-test code that references test-only helpers still has to compile. Keep test helpers inside the test blocks.
- **Stringification is opt-in.** Operands without an `operator<<` or a `doctest::toString`/`StringMaker` specialization print as `{?}` on failure. For custom types you must provide one to get useful diagnostics.
- **Subcase re-execution can double-run expensive setup.** Since the case restarts per subcase leaf, heavyweight fixture work placed before subcases runs once per leaf. For costly shared state prefer `TEST_CASE_FIXTURE` or hoist the setup.
- **No test-level parallelism built in.** There is no sharded/parallel runner analogous to `gtest-parallel`; large suites parallelize at the CI/process level instead.
- **Maintenance cadence is uneven.** After 2.4.11 (March 2023) the project was quiet through a single 2.4.12 patch (April 2025) before a burst of 2.5.x releases in 2026[^5]. The framework is stable and API-conservative rather than fast-moving; plan for infrequent releases.

## When to Use / When Not

**Use when:**
- Compile-time and iteration speed matter and you have (or want) many test files or thousands of assertions.
- You want tests to live beside production code, or to ship a build with tests optionally compiled out.
- You want a Catch2-style API with a lighter header and zero warnings on strict compiler settings.
- You need asserts usable outside a test context or from threads inside a test.

**Avoid when:**
- You need first-class mocking, death tests, or a large matcher library out of the box — GoogleTest/GoogleMock is the fuller ecosystem.
- You depend on built-in parallel/sharded test execution for a very large suite.
- You are on a pre-C++11 toolchain (the `1.2.9` tag is the last C++98-capable version).

## Alternatives

- catchorg/Catch2 — the framework doctest is modeled on; richer feature set and matchers, but heavier compile times. Use it when breadth beats compile speed.
- google/googletest — the de-facto standard with GoogleMock, death tests, and huge tooling support. Use it when you need mocking or maximal ecosystem integration.
- boost/test — deep, mature, tightly integrated with Boost. Use it when you already live in Boost and want its fixtures/data-driven tests.
- cpputest/cpputest — includes a memory-leak detector and a C-friendly mocking layer. Use it for embedded/C-heavy codebases.
- catchorg/Catch2 v2 single-header — if you specifically want single-file Catch semantics and don't mind the compile cost.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2016-05-22 | Initial public release[^1]. |
| 1.2.9 | 2018-05-10 | Last C++98-capable tag; later versions require C++11. |
| 2.0.0 | 2018-08-23 | Major line; API cleanups, drops C++98 support. |
| 2.3.0 | 2019-03-23 | Thread-safe asserts, reporter improvements. |
| 2.4.0 | 2020-06-27 | Long-lived 2.4.x series begins. |
| 2.4.11 | 2023-03-15 | Last release before an extended quiet period[^5]. |
| 2.4.12 | 2025-04-28 | Lone patch during the lull. |
| 2.5.0 | 2026-03-27 | Release activity resumes; 2.5.x cadence through 2026. |
| 2.5.3 | 2026-07-06 | Latest release at time of writing[^6]. |

## References

[^1]: doctest release history, tag `1.0.0` (2016-05-22). https://github.com/doctest/doctest/releases
[^2]: Compile-time and runtime benchmarks maintained in-repo. https://github.com/doctest/doctest/blob/master/doc/markdown/benchmarks.md
[^3]: Configuration reference — `DOCTEST_CONFIG_DISABLE`. https://github.com/doctest/doctest/blob/master/doc/markdown/configuration.md
[^4]: FAQ — thread awareness of asserts. https://github.com/doctest/doctest/blob/master/doc/markdown/faq.md
[^5]: Release list showing the 2.4.11 (2023-03) → 2.4.12 (2025-04) → 2.5.0 (2026-03) gap. https://github.com/doctest/doctest/releases
[^6]: doctest `v2.5.3`, published 2026-07-06. https://github.com/doctest/doctest/releases/tag/v2.5.3

## Tags

cpp, c-plus-plus, testing-framework, unit-testing, header-only, single-file, tdd, compile-time, catch2-alternative, cpp11
