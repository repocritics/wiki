# nunit/nunit

> The unit-testing framework for .NET languages — the constraint-based assertion model and the oldest continuously maintained xUnit descendant in the ecosystem.

[GitHub repo](https://github.com/nunit/nunit) ·
[Official website](https://nunit.org/) ·
[License: MIT](https://raw.githubusercontent.com/nunit/nunit/main/LICENSE.txt)

## Overview

NUnit is a unit-testing framework for all .NET languages (C#, F#, VB.NET), runnable on Windows, macOS, and Linux. It began as a direct port of JUnit to .NET in the early 2000s and has been rewritten more than once since; the modern line starts at NUnit 3.0 (2015), which discarded the NUnit 2.x engine and internals for a ground-up redesign[^1]. It is deliberately non-opinionated and broad: it offers many overlapping ways to express the same assertion and exposes numerous extension points, which is both its appeal and the reason its surface area is large.

The defining design choice is the **constraint-based assertion model** — `Assert.That(actual, Is.EqualTo(expected))` — a fluent, composable grammar of constraints (`Is`, `Has`, `Does`, `Contains`, `Throws`) that reads left-to-right and produces structured failure messages. The older "classic" model (`Assert.AreEqual`, `Assert.IsTrue`) still exists but was demoted in NUnit 4[^2]. Alongside this, NUnit is attribute-driven: `[Test]`, `[TestFixture]`, `[TestCase]`, `[SetUp]`, `[TearDown]`, and parameterized-test attributes like `[Values]` and `[TestCaseSource]` define behavior declaratively.

The most important structural fact for newcomers is that **this repository is only the framework** — the library you reference to *write* tests. Actually *running* them requires a separate runner: `NUnit3TestAdapter` under `dotnet test` / Visual Studio Test Explorer, or the standalone `nunit-console` engine. The framework, adapter, console, and Roslyn analyzers live in different repositories and version independently[^3]. This split is the single most common source of "why won't my tests run" confusion.

## Getting Started

```bash
dotnet new nunit -o MyProject.Tests   # scaffolds framework + adapter + Test.Sdk
cd MyProject.Tests
dotnet test
```

Or add the pieces to an existing test project manually (all three are required to run under `dotnet test`):

```bash
dotnet add package NUnit
dotnet add package NUnit3TestAdapter
dotnet add package Microsoft.NET.Test.Sdk
```

```csharp
using NUnit.Framework;

[TestFixture]
public class CalculatorTests
{
    [TestCase(2, 3, 5)]
    [TestCase(-1, 1, 0)]
    public void Add_ReturnsSum(int a, int b, int expected)
    {
        Assert.That(a + b, Is.EqualTo(expected));
    }

    [Test]
    public void Divide_ByZero_Throws()
    {
        Assert.That(() => 1 / int.Parse("0"), Throws.TypeOf<DivideByZeroException>());
    }
}
```

## Architecture / How It Works

NUnit discovers tests by **reflection**: the adapter or engine loads the compiled test assembly, walks types for `[TestFixture]` (or plain classes containing `[Test]` methods since attribute-free fixtures are allowed), and builds a tree of test cases. Parameterized attributes (`[TestCase]`, `[TestCaseSource]`, `[Values]`, `[ValueSource]`, `[Combinatorial]`) expand at discovery time into individual cases, so a single method can materialize as dozens of tree nodes.

**Fixture lifecycle** is NUnit's biggest behavioral departure from xUnit.net. By default NUnit creates **one instance of the fixture class per fixture**, reused across all its test methods — not a fresh instance per test. Shared mutable fields therefore leak state between tests unless reset in `[SetUp]`. Since NUnit 3.13 you can opt into per-test-case isolation with `[FixtureLifeCycle(LifeCycle.InstancePerTestCase)]`, which is the safer default many teams now apply assembly-wide.

**Assertion model.** The constraint model is the primary API: constraints are objects implementing `IConstraint`, combined with `.And` / `.Or` / `Is.Not`, and applied by `Assert.That`. `Assert.Multiple(() => { ... })` defers failures so several assertions report together instead of stopping at the first. The classic static asserts were physically moved to a separate `ClassicAssert` class in NUnit 4 — a source-breaking change that broke essentially every NUnit 3 test file using `Assert.AreEqual` until re-pointed[^2].

**Parallelism** is opt-in via `[Parallelizable]` (at assembly, fixture, or method scope) and `[LevelOfParallelism(n)]`. NUnit parallelizes across fixtures/tests but shares one process; static state, `Console` redirection, and non-thread-safe SUTs are the usual failure points. `[Apartment]`, `[Timeout]`, and `[Retry]` control threading and flakiness handling.

The framework targets `netstandard2.0` plus specific .NET / .NET Framework TFMs, so it runs on both the legacy Framework and modern .NET runtimes from one package.

## Production Notes

**The three-package requirement is a real trap.** Referencing `NUnit` alone compiles tests but discovers zero of them under `dotnet test` or VS Test Explorer — `NUnit3TestAdapter` and `Microsoft.NET.Test.Sdk` must both be present. CI logs showing "no tests found" almost always mean a missing adapter, not a broken test.

**Shared fixture state causes order-dependent flakiness.** Because the default lifecycle reuses one fixture instance, a test that mutates a field and a later test that reads it will pass in isolation and fail (or pass spuriously) depending on run order and parallelism. Standardizing on `InstancePerTestCase` or disciplined `[SetUp]` resets avoids this.

**NUnit 4 upgrade is not mechanical.** Beyond `ClassicAssert`, the 4.0 release tightened nullable-reference annotations, removed long-deprecated APIs, and dropped older target frameworks[^2]. Teams typically run the NUnit analyzers to auto-migrate classic asserts to the constraint model before upgrading. Read the official breaking-changes and migration guides first — the 3→4 jump changed public API surface, not just internals.

**Async assertions have sharp edges.** `Assert.That` accepts async delegates and there is `Assert.MultipleAsync`, but forgetting to await, or asserting on a `Task` rather than its result, silently tests the wrong thing. The NUnit analyzers catch many of these at build time and are worth enabling.

**Test isolation is your responsibility.** NUnit does not sandbox tests; static singletons, `DateTime.Now`, ambient culture, and the working directory are shared. `[SetCulture]`, `[SetUICulture]`, and `TestContext.CurrentContext` help, but there is no per-test process isolation as some other stacks offer.

## When to Use / When Not

**Use when:**
- You want the richest built-in assertion grammar and the widest set of parameterization attributes (`[TestCaseSource]`, `[Combinatorial]`, `[Range]`, `[Sequential]`).
- You maintain older .NET Framework code — NUnit's `netstandard2.0` reach and long history make it a safe target across runtimes.
- You value explicit `[SetUp]`/`[TearDown]`/`[OneTimeSetUp]` lifecycle hooks over constructor-based setup.
- You need integration/system tests, not just units — retries, timeouts, and apartment control are first-class.

**Avoid when:**
- You want enforced per-test isolation by default — xUnit.net's new-instance-per-test model gives that without opt-in.
- You prefer a small, opinionated surface: NUnit's many redundant assertion styles are a lot to standardize a team around.
- You're on a brand-new .NET-only codebase where the ecosystem default (xUnit) is already what your templates and CI assume.

## Alternatives

- xunit/xunit — new fixture instance per test (isolation by default), constructor/`IDisposable` setup instead of attributes; the common default for greenfield .NET. Use when you want enforced isolation and a leaner API.
- microsoft/testfx — MSTest, Microsoft's own framework; use when you want the vendor-supported option tightly wired into Visual Studio and Azure DevOps.
- fluentassertions/fluentassertions — assertion library layered on top of any framework; use with NUnit for more readable assertions (note: v8+ moved to a commercial license).
- shouldly/shouldly — MIT-licensed fluent assertion library; use as a lighter, free alternative to FluentAssertions alongside any runner.
- nunit/nunit-console — the standalone runner/engine for this framework; use when you need to run NUnit tests outside `dotnet test` or VS.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.0 | 2002 | Early JUnit-derived line; long-lived 2.x series. |
| 3.0 | 2015-11 | Ground-up rewrite; new engine, constraint model, parallelism[^1]. |
| 3.13 | 2021 | `[FixtureLifeCycle]` for per-test-case instances. |
| 4.0 | 2023 | `ClassicAssert` split, nullable annotations, dropped old TFMs[^2]. |
| 5.0 | recent | Modernized line targeting newer .NET / C# features (per README)[^4]. |

## References

[^1]: NUnit documentation, "Release Notes / Framework" — history of the NUnit 3 rewrite. https://docs.nunit.org/articles/nunit/release-notes/framework.html
[^2]: NUnit documentation, "Breaking Changes" and "NUnit 4.0 Migration Guide". https://docs.nunit.org/articles/nunit/release-notes/breaking-changes.html and https://docs.nunit.org/articles/nunit/release-notes/Nunit4.0-MigrationGuide.html
[^3]: NUnit README, "NUnit Projects" — framework, VS adapter, analyzers, and console/engine are separate repositories. https://github.com/nunit/nunit
[^4]: NUnit README — "The latest version, version 5, is an upgrade from ... NUnit 3 and NUnit 4." https://github.com/nunit/nunit

## Tags

csharp, dotnet, testing, unit-testing, test-framework, tdd, nunit, assertions, fsharp, dotnet-testing
