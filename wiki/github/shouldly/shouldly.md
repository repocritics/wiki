# shouldly/shouldly

> Assertion library for .NET whose entire reason to exist is the failure message, not the API surface.

[GitHub repo](https://github.com/shouldly/shouldly) ·
[Documentation](https://docs.shouldly.org) ·
[License: BSD-3-Clause](https://github.com/shouldly/shouldly/blob/master/LICENSE.txt)

## Overview

Shouldly is a small assertion library for .NET, first created in 2010[^1]. It is not a test framework: it plugs into xUnit, NUnit, MSTest, or anything else that reports a thrown exception as a failure. Its scope is deliberately narrow — replace `Assert.AreEqual(expected, actual)` with `actual.ShouldBe(expected)` — and its selling point is what happens when the assertion fails.

The defining trick is that Shouldly reports the *expression that was asserted*, not just the values. `Assert.That(contestant.Points, Is.EqualTo(1337))` fails with "Expected 1337 but was 0"; `contestant.Points.ShouldBe(1337)` fails with "contestant.Points should be 1337 but was 0". Recovering that `contestant.Points` text is the whole engineering problem the library solves, and it is also where every one of Shouldly's production caveats comes from.

As of 2026 the project has roughly 3.4k stars and 420 forks — modest by framework standards, but it is a mature, single-purpose library that most teams install once and forget. It is still actively maintained (commits within the current week) and remains widely used in .NET test suites, though it has ceded mindshare to FluentAssertions over the years. Shouldly's counter-argument has always been terseness: one method per intent (`ShouldBe`, `ShouldContain`, `ShouldThrow`) rather than chained `.Should().Be().And()...` expressions.

## Getting Started

```bash
dotnet add package Shouldly
```

```csharp
using Shouldly;
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Adds_two_numbers()
    {
        var result = 2 + 2;

        result.ShouldBe(4);                       // scalar equality
        result.ShouldBeGreaterThan(0);            // comparisons
        new[] { 1, 2, 3 }.ShouldContain(2);       // collections

        Should.Throw<DivideByZeroException>(() => { var _ = 1 / int.Parse("0"); });

        // group assertions so ALL failures report at once, not just the first
        result.ShouldSatisfyAllConditions(
            () => result.ShouldBePositive(),
            () => result.ShouldBe(4)
        );
    }
}
```

There is no runner to configure and no `[Should]` attribute — the extension methods live on every type via `using Shouldly;`.

## Architecture / How It Works

Shouldly is a set of extension methods (`ShouldBe`, `ShouldNotBe`, `ShouldContain`, `ShouldThrow`, `ShouldBeOfType`, …) that throw a `ShouldAssertException` when the condition is false. The interesting part is how the message is built.

To print `contestant.Points should be 1337`, Shouldly needs the literal source text of the argument you passed. Historically it obtained this by walking the stack trace to find the calling frame, reading the corresponding `.cs` file off disk, and parsing the line to extract the expression — which required both the source files and debug symbols (PDBs) to be present at test-execution time. Newer versions lean on the C# compiler feature `[CallerArgumentExpression]`, which captures the argument text at compile time and removes the need to read source at runtime[^2]. This is the single most important thing to understand about the library: the quality of its output is a function of how much caller information the runtime can recover, and that varies by build configuration.

Beyond messaging, Shouldly ships a handful of assertion categories worth knowing:

- **`ShouldSatisfyAllConditions`** — evaluates every nested assertion and aggregates failures, instead of stopping at the first. This is the closest thing to a "soft assertions" mode.
- **`ShouldBeEquivalentTo`** — reflection-based deep/structural equality, for comparing object graphs without overriding `Equals`.
- **`ShouldMatchApproved`** — approval testing: compares output against a checked-in "approved" file. To get a visual diff on mismatch you must add the separate `Shouldly.DiffEngine` package and call `ShouldMatchConfiguration.ShouldMatchApprovedDefaults.ConfigureDiffEngine()`[^3].
- **`ShouldCompleteIn`** — asserts an operation finishes within a timeout.
- Async variants (`Should.ThrowAsync`, `Should.CompleteInAsync`) for awaited code.

Equality itself defers to the type's own `Equals`/`IEquatable`/`IComparable`, so `ShouldBe` behaves exactly as your type does — including all the surprises of value vs. reference equality.

## Production Notes

- **Messages degrade when caller info is unavailable.** In build/CI configurations without source files or PDBs alongside the test assembly — some release pipelines, certain container layouts, trimmed/single-file publishes — the "print the expression" magic falls back to a generic message. The assertions still pass/fail correctly; you just lose the feature you installed the library for. If a colleague reports "Shouldly messages are worse on CI than locally," this is almost always why.
- **Collection order matters by default.** `ShouldBe` on an `IEnumerable` compares element-by-element in order. To compare as sets, pass the ignore-order overload (`ShouldBe(expected, ignoreOrder: true)`); otherwise a reordered-but-equal collection fails.
- **Floating point needs a tolerance.** `ShouldBe` on `double`/`float` has an overload taking a tolerance argument. Using the plain overload subjects you to normal binary floating-point inequality.
- **`ShouldBeEquivalentTo` is reflection-based.** It is convenient but comparatively slow on large graphs and has historically had edge cases around cycles, dictionaries, and anonymous types. For heavy structural comparison, snapshot tools are often a better fit.
- **Approval testing carries a second dependency and a workflow.** `ShouldMatchApproved` writes a `.received` file on first run that you must review and rename to `.approved`; forgetting to commit approved files makes CI red. The diff viewer only appears with `Shouldly.DiffEngine` configured.
- **Major-version upgrades tightened target frameworks.** Shouldly 4.x dropped older/legacy target frameworks and modernized internals (including the caller-expression path). Projects on very old .NET Framework versions may need to pin an earlier Shouldly release.
- **It is assertions only.** No test discovery, no fixtures, no parallelism control — pair it with a runner. It also does not replace mocking (Moq/NSubstitute) or data-driven test attributes.

## When to Use / When Not

**Use when:**
- You want readable failure messages with minimal ceremony and one method per assertion intent.
- Your test suite already uses xUnit/NUnit/MSTest and you only want to improve assertions.
- You prefer `value.ShouldBe(x)` terseness over long fluent chains.
- You want lightweight approval testing (`ShouldMatchApproved`) without adopting a full snapshot framework.

**Avoid when:**
- You build/deploy tests in configurations that strip source and symbols and cannot change that — you lose the headline feature.
- You want an expansive fluent DSL with rich chained matchers and extensive collection/exception assertions — FluentAssertions/AwesomeAssertions cover more surface.
- You need serious structural/snapshot comparison of large object graphs — a dedicated snapshot tool is more capable than `ShouldBeEquivalentTo`.

## Alternatives

- fluentassertions/fluentassertions — larger fluent matcher API; note its license became commercial for v8+ (2025), which pushed many OSS users elsewhere.
- meziantou/AwesomeAssertions — community OSS fork of FluentAssertions' last MIT release; use when you want the FluentAssertions API without the commercial license.
- nunit/nunit — its `Assert.That` + constraint model is a built-in alternative if you already run NUnit and want no extra dependency.
- xunit/xunit — the built-in `Assert.*` class is enough for simple suites where message quality is not a priority.
- VerifyTests/Verify — purpose-built snapshot/approval testing; use instead of `ShouldMatchApproved` when approvals are your main need.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010-08 | Repository created; extension-method assertions with source-derived messages[^1]. |
| 3.x | 2018–2020 | Long-lived line across the .NET Core transition; broad target-framework support. |
| 4.x | 2021–2022 | Major release: dropped legacy targets, modernized internals and the caller-expression path[^2]. |

Precise per-release dates beyond the above are best confirmed against the NuGet version history[^4].

## References

[^1]: shouldly/shouldly repository (created 2010-08-23). https://github.com/shouldly/shouldly
[^2]: `CallerArgumentExpressionAttribute`, used to capture asserted expressions at compile time. https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.callerargumentexpressionattribute
[^3]: Shouldly README — `ShouldMatchApproved` and `Shouldly.DiffEngine` setup. https://github.com/shouldly/shouldly#installation
[^4]: Shouldly on NuGet (version history and download stats). https://www.nuget.org/packages/Shouldly

## Tags

dotnet, csharp, testing, assertions, unit-testing, test-framework, xunit, nunit, fluent-api, approval-testing
