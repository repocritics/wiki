# xunit/xunit

> The unit-testing framework that enforces test isolation by convention — fresh class instance per test, parallel by default, and no setup/teardown attributes.

[GitHub repo](https://github.com/xunit/xunit) ·
[Official website](https://xunit.net/) ·
[License: Apache-2.0](https://github.com/xunit/xunit/blob/main/LICENSE)

## Overview

xUnit.net is a unit-testing framework for .NET (C#, F#, Visual Basic), created by Brad Wilson and Jim Newkirk — the latter an original author of NUnit — as a deliberate rethink of what a .NET test framework should enforce[^1]. Where NUnit and MSTest inherited the JUnit `[SetUp]`/`[TearDown]` lifecycle, xUnit.net dropped it: the test class constructor is your setup, `IDisposable.Dispose` is your teardown, and the runner constructs a brand-new instance of the test class for every single test method. That one decision — isolation by construction rather than by discipline — is the framework's defining opinion, and the reason teams either love it or find it pedantic.

It is one of the three mainstream .NET test frameworks (alongside NUnit and MSTest) and is the default in a large share of modern C# projects, including much of the .NET runtime's own test suite. It is part of the .NET Foundation and governed by a Project Lead rather than a company[^2].

The current line, v3, is a substantial architectural change from the long-lived 2.x series. As of 2026 the project is actively maintained — commits land continuously and a v3 4.0 pre-release train is in flight — but it carries the migration cost of that rewrite, which is the main thing a v2 shop needs to plan for.

## Getting Started

```bash
# v3: create a test project (compiles to a standalone executable)
dotnet new install xunit.v3.templates
dotnet new xunit3 -o MyTests
cd MyTests
dotnet test
```

```csharp
using Xunit;

public class CalculatorTests
{
    // [Fact] — a single, parameterless test
    [Fact]
    public void Adds_two_numbers()
    {
        Assert.Equal(4, 2 + 2);
    }

    // [Theory] — one test body, many data rows
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(-1, 1, 0)]
    public void Adds_various(int a, int b, int expected)
    {
        Assert.Equal(expected, a + b);
    }
}
```

Per-test setup goes in the constructor; cleanup goes in `Dispose()`. There is no `[SetUp]`.

## Architecture / How It Works

The runtime unit of isolation is the **test class instance**. For every `[Fact]`/`[Theory]` case, the runner news up the class, runs the one method, then disposes it. Shared state between tests therefore has to be explicit, via fixtures:

- **`IClassFixture<T>`** — one instance of `T` shared across all tests in a class (expensive setup done once).
- **`ICollectionFixture<T>`** — shared across all classes in a named `[Collection]`.

**Parallelism is on by default.** In 2.x and v3, each test class is its own "collection" unless annotated, and collections run concurrently[^3]. Tests *within* a class run sequentially, but two different classes run at the same time. This is the single most common source of "it passes alone, fails in the suite" bugs when tests touch shared files, static state, or a database.

Tests are parameterized only through `[Theory]` data sources — `[InlineData]`, `[MemberData]`, `[ClassData]`. There is no `[ExpectedException]`; exceptions are asserted with `Assert.Throws<T>`. Console output is not captured — a test must take an injected `ITestOutputHelper` to record diagnostics.

The **assertion library lives in a separate repository** (`xunit/assert.xunit`) and is shared into the main library as source rather than a binary dependency, which is why contributing to assertions has its own workflow[^1]. A companion Roslyn analyzer package (`xunit.analyzers`) ships with the framework and flags common misuse (async-void tests, non-serializable theory data, misused assertions) at compile time.

The biggest structural change in **v3** is that a test project now compiles to a **standalone executable** instead of a library that a separate runner process loads and reflects over[^4]. Each test assembly is its own program, and it plugs into either the **Microsoft Testing Platform (MTP)** or the older VSTest protocol. v3 also consolidated the previously fragmented package set (`xunit.core`, `xunit.execution`, `xunit.abstractions`, …) and targets .NET 8.0+ or .NET Framework 4.7.2+[^1].

## Production Notes

- **Default parallelism is a footgun for integration tests.** Any suite that hits a shared database, filesystem, or static singleton will see nondeterministic failures until you serialize the offending classes into a single `[Collection]` or disable parallelization (`[assembly: CollectionBehavior(DisableTestParallelization = true)]`). This is the number-one migration surprise for teams coming from MSTest's serial default.
- **Constructor-per-test repeats expensive setup.** Because the class is reconstructed for every method, any heavy work in the constructor (spinning up a container, seeding a DB) runs once *per test*. Move it into an `IClassFixture` or `ICollectionFixture`, or the suite crawls.
- **`async void` tests silently do nothing meaningful.** Test methods must be `async Task`; the analyzer catches this, but only if `xunit.analyzers` is present (it is, by default, with the metapackage).
- **Non-serializable `[MemberData]` collapses in test explorers.** If theory data isn't serializable, Visual Studio / VS Code show one aggregated test case instead of a row per input, and reruns of a single case don't work. Prefer primitives or implement `IXunitSerializable`.
- **v2 → v3 is a real migration, not a version bump.** The package identity changes (`xunit` → `xunit.v3`), the assembly-as-executable model changes how CI invokes tests, and some extensibility APIs (`ITestFrameworkDiscoverer`, custom attributes) were reshaped. Budget time; don't treat it as a `dotnet add package` upgrade[^4].
- **License metadata is ambiguous on GitHub.** GitHub's license detector reports `NOASSERTION` for the repo, but the project documents itself as Apache-2.0 (OSI-approved)[^1]. Treat it as Apache-2.0; verify the bundled `LICENSE` if you have strict compliance tooling.

## When to Use / When Not

**Use when:**
- You want test isolation enforced by the framework rather than by reviewer discipline.
- You're on modern .NET and want parallel test execution and first-class Microsoft Testing Platform support out of the box.
- You're contributing to or aligning with the broader .NET open-source ecosystem, where xUnit is the common default.

**Avoid when:**
- Your team relies heavily on `[SetUp]`/`[TearDown]`, one-time class fixtures with attribute lifecycle, or NUnit-style `[TestCase]` ergonomics — porting muscle memory is friction.
- You need the officially Microsoft-supported-and-shipped framework for a conservative enterprise mandate (that's MSTest).
- You have a large serial integration suite and don't want to reason about opt-out parallelism.

## Alternatives

- nunit/nunit — use instead when you want the classic `[SetUp]`/`[TearDown]` lifecycle, `[TestCase]` inline parameterization, and a broader built-in assertion and constraint set.
- microsoft/testfx — MSTest; use when you want the framework Microsoft ships and supports directly, tightly integrated with Visual Studio.
- fluentassertions/fluentassertions — complementary assertion library layered on top of any framework (note: v8+ moved to a commercial license for some uses).
- shouldly/shouldly — permissively licensed assertion library with readable failure messages, a drop-in for `Assert.*` calls.
- fixie/fixie — use when you want convention-based test discovery you define yourself, rather than attribute-driven `[Fact]`/`[Theory]`.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2007 | Initial release by Brad Wilson and Jim Newkirk; constructor/`Dispose` lifecycle, no `[SetUp]`[^1]. |
| 2.0 | 2015-03 | Major rewrite; parallel execution by default, new extensibility model, `[Theory]` data sources[^3]. |
| 2.4.x | 2018–2023 | Long-lived stable line; widely adopted default across .NET projects. |
| 2.9.x | 2024 | Late 2.x maintenance releases alongside early v3 work. |
| v3 (1.0) | 2025 | Test-assembly-as-executable model, Microsoft Testing Platform support, package consolidation, .NET 8+ / .NET FX 4.7.2+[^4]. |

## References

[^1]: xUnit.net repository README — project scope, packages, Apache-2 licensing, and separate assertion-library workflow. https://github.com/xunit/xunit
[^2]: xUnit.net governance — Project Lead model under the .NET Foundation. https://xunit.net/governance
[^3]: xUnit.net docs, "Running tests in parallel" — collection-based parallelism, default since 2.0. https://xunit.net/docs/running-tests-in-parallel
[^4]: xUnit.net docs, "What's new in v3" / getting started (v3) — executable test projects and Microsoft Testing Platform. https://xunit.net/docs/getting-started/v3/getting-started

## Tags

csharp, dotnet, unit-testing, test-framework, testing, fsharp, tdd, assertions, parallel-testing, xunit
