# ThrowTheSwitch/Unity

> Unit testing for C in one .c file and two headers — built for code that has to run on microcontrollers, not just on your laptop.

[GitHub repo](https://github.com/ThrowTheSwitch/Unity) ·
[Official website](https://throwtheswitch.org) ·
[License: MIT](https://github.com/ThrowTheSwitch/Unity/blob/master/LICENSE.txt)

## Overview

Unity Test (no relation to the game engine) is a unit testing framework for C maintained by ThrowTheSwitch.org — Mike Karlesky, Mark VanderVoord, and Greg Williams. The project dates to 2007[^1], with the GitHub repo created in January 2012. Its design constraint is embedded reality: the entire core is `unity.c` plus `unity.h` and `unity_internals.h`, with no dynamic allocation, no required OS, and output funneled through a single character-out function so results can be streamed over a UART from a bare-metal target[^2]. It works with any C compiler and any build system; ThrowTheSwitch's own Ceedling wraps it in a Ruby-based build tool if you want the batteries included.

The defining tension is that C has no reflection, so Unity cannot discover tests. You either hand-write a `main()` that calls `RUN_TEST` for each test function, or you use the bundled Ruby script (`auto/generate_test_runner.rb`) to generate the runner — which quietly reintroduces a scripting-language dependency into a "pure C" framework. Most serious users accept the Ruby layer, and usually the full ThrowTheSwitch stack: Unity for assertions, CMock for generated mocks, Ceedling to orchestrate both.

At 5.3k stars it looks mid-sized, but GitHub stars undercount embedded tooling: Unity is the test framework shipped inside Espressif's ESP-IDF[^3] and the default framework in PlatformIO's unit testing system[^4], which puts it in front of a large fraction of hobbyist and commercial MCU firmware.

## Getting Started

Vendor the three core files (or add the repo as a submodule/CMake FetchContent) — there is no package to install for the C core.

```c
// test_calculator.c
#include "unity.h"
#include "calculator.h"

void setUp(void) {}      // runs before each test
void tearDown(void) {}   // runs after each test

void test_add_returns_sum(void) {
    TEST_ASSERT_EQUAL_INT(7, add(3, 4));
}

void test_add_handles_negatives(void) {
    TEST_ASSERT_EQUAL_INT(-1, add(2, -3));
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_add_returns_sum);
    RUN_TEST(test_add_handles_negatives);
    return UNITY_END();
}
```

```bash
gcc -Isrc src/unity.c calculator.c test_calculator.c -o test_calculator
./test_calculator
# test_calculator.c:10:test_add_returns_sum:PASS
```

## Architecture / How It Works

The assertion macros (`TEST_ASSERT_EQUAL_INT`, `TEST_ASSERT_EQUAL_HEX8_ARRAY`, and roughly a hundred siblings) are thin wrappers that pass file/line and expected/actual values into a small set of `UnityAssert*` functions in `unity.c`. Keeping the logic in functions rather than macro bodies keeps code size down and means arguments are evaluated once — relevant when your test asserts on a register read with side effects.

Failure handling uses `setjmp`/`longjmp`: `RUN_TEST` wraps each test in `TEST_PROTECT()`, and a failing assertion longjmps back to the runner so the remaining tests still execute. On targets where `setjmp` is unavailable or prohibited (common under MISRA-style rules), `UNITY_EXCLUDE_SETJMP_H` swaps this for a plain return — with the documented limitation that a failed assertion inside a helper function can no longer unwind out of the test[^5].

Everything else is compile-time configuration via `-D` flags or a `unity_config.h`[^5]: exclude float/double support entirely (`UNITY_EXCLUDE_FLOAT` — many MCUs have no FPU), set integer and pointer widths for odd architectures, and redirect output by defining `UNITY_OUTPUT_CHAR` to your UART putc. Test reports are a fixed machine-parseable format (`file:line:test:PASS|FAIL:message`), which is what Ceedling, PlatformIO, and various CI plugins scrape.

The repo also carries `extras/`: `unity_fixture` adds CppUTest-style `TEST_GROUP` organization and leak-checking hooks, and `unity_memory` provides malloc instrumentation. Parameterized tests (`TEST_CASE`, `TEST_RANGE`) exist but only function through the Ruby-generated runner — the plain-C path doesn't support them.

## Production Notes

**The Ruby dependency is de facto, not optional.** Hand-maintaining runners is viable for a handful of test files and miserable beyond that. Budget for Ruby in your CI image if you adopt Unity at scale, or adopt Ceedling and accept its conventions wholesale. CMake users frequently write their own runner-generation glue instead.

**On-target testing needs plumbing.** Unity prints results; it does not transport them. Running tests on real hardware means wiring `UNITY_OUTPUT_CHAR` to a UART/semihosting channel and having something on the host side parse the stream and decide pass/fail. PlatformIO and ESP-IDF have this glue built; roll-your-own setups underestimate it.

**Struct and float assertions have sharp edges.** `TEST_ASSERT_EQUAL_MEMORY` on structs compares padding bytes, so uninitialized padding produces flaky failures — compare fields individually or memset first. `TEST_ASSERT_EQUAL_FLOAT` is not exact equality; it passes within a small relative tolerance (`UNITY_FLOAT_PRECISION`), which is usually what you want and occasionally a silent surprise.

**Release cadence is glacial; master is alive.** v2.5.2 (Jan 2021) to v2.6.0 (Mar 2024) was over three years[^6], yet the repo saw pushes as recently as July 2026 with v2.7.0 tagged the same day. Read this as API stability by policy rather than abandonment — but if you pin releases, fixes can take years to reach you, and many downstreams (ESP-IDF included) vendor specific commits instead.

**No isolation, no parallelism.** Tests run sequentially in one process sharing one global Unity state struct. A test that hard-faults the MCU takes the whole run down with it — there is no fork-based sandboxing as in `check`. On-target suites typically mitigate with a watchdog plus a host-side runner that reflashes and resumes.

**Mocking is a separate purchase.** Unity has assertions only. Interaction testing against HALs and drivers means adding CMock (generated mocks from headers, more Ruby) or hand-rolled fakes via the linker.

## When to Use / When Not

**Use when:**
- You are testing C targeting microcontrollers, and tests must compile with the cross-toolchain and run on-target or in a simulator.
- You need a framework with zero heap usage, no OS assumptions, and C89-adjacent portability.
- You want the Ceedling/CMock ecosystem, or you are already inside ESP-IDF or PlatformIO where Unity is the paved road.
- You want assertion failure output that machines can parse without configuration.

**Avoid when:**
- Your codebase is C++ or can be tested from C++ — google/googletest or cpputest/cpputest give you fixtures, matchers, and parameterized tests without Ruby glue.
- You want built-in mocking in one tool — CppUTest includes it; Unity requires CMock alongside.
- You are testing hosted (Linux/POSIX) C and want crash isolation or parallel execution — fork-based frameworks handle a segfaulting test gracefully; Unity does not.
- You need rich test discovery and IDE integration out of the box; Unity's no-reflection design pushes that onto external tooling.

## Alternatives

- cpputest/cpputest — use instead when a C++ compiler is available for the test build and you want mocking plus memory-leak detection built into the framework.
- google/googletest — use instead for C++-first projects or C tested through a C++ harness on hosted platforms.
- libcheck/check — use instead for POSIX-hosted C where fork-based test isolation (a segfault fails one test, not the run) matters.
- silentbicycle/greatest — use instead when even Unity is too heavy: a single header, no runner generation, fewer assertion types.
- ThrowTheSwitch/CMock — not an alternative but the companion: generated mocks for Unity; evaluate them together.

## History

| Version | Date | Notes |
|---------|------|-------|
| origins | 2007 | Framework created by Karlesky/VanderVoord/Williams for embedded consulting work[^1]. |
| GitHub | 2012-01-26 | Repo published on GitHub. |
| v2.4.0 | 2016-10-28 | First tagged GitHub release of the 2.4 line[^6]. |
| v2.5.0 | 2019-10-30 | Restructured release; last minor line before a multi-year gap. |
| v2.5.2 | 2021-01-29 | Final 2.5 patch; start of a 3-year release quiet period. |
| v2.6.0 | 2024-03-10 | First release in over three years[^6]. |
| v2.6.1 | 2025-01-01 | Patch release. |
| v2.7.0 | 2026-07-16 | Current release; tagged the same day as latest repo push. |

## References

[^1]: Unity README, copyright line "2007 - 2026 Unity Project by Mike Karlesky, Mark VanderVoord, and Greg Williams". https://github.com/ThrowTheSwitch/Unity/blob/master/README.md
[^2]: Unity Getting Started Guide. https://github.com/ThrowTheSwitch/Unity/blob/master/docs/UnityGettingStartedGuide.md
[^3]: ESP-IDF documentation, "Unit Testing in ESP32" — built on Unity. https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/unit-tests.html
[^4]: PlatformIO documentation, "Unit Testing" — Unity as default framework. https://docs.platformio.org/en/latest/advanced/unit-testing/index.html
[^5]: Unity Configuration Guide (`UNITY_EXCLUDE_SETJMP_H`, `UNITY_OUTPUT_CHAR`, float exclusion). https://github.com/ThrowTheSwitch/Unity/blob/master/docs/UnityConfigurationGuide.md
[^6]: GitHub releases for ThrowTheSwitch/Unity (v2.4.0 2016-10-28 … v2.7.0 2026-07-16). https://github.com/ThrowTheSwitch/Unity/releases

## Tags

c, unit-testing, embedded, microcontroller, test-framework, assertions, bare-metal, tdd, firmware, single-file-library
