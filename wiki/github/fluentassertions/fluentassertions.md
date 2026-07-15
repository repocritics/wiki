# fluentassertions/fluentassertions

> Extension methods that turn .NET test assertions into readable, chainable statements — now under a commercial license from v8 onward.

[GitHub repo](https://github.com/fluentassertions/fluentassertions) ·
[Official website](https://fluentassertions.com) ·
[License: NOASSERTION (Xceed Community License, v8+)](https://github.com/fluentassertions/fluentassertions/blob/main/LICENSE)

## Overview

Fluent Assertions is a .NET assertion library that replaces framework-native asserts (`Assert.AreEqual`, `Assert.Equal`, `Assert.That`) with a fluent, discoverable API: `result.Should().Be(42)`, `collection.Should().ContainSingle()`, `action.Should().Throw<ArgumentException>()`. Its selling point is failure messages — instead of "Expected: 4, But was: 3" it produces "Expected numbers to contain 4 item(s) because we thought we put four items in the collection, but found 3." It was originally authored by Dennis Doomen, with Jonas Nyrup as a longtime co-maintainer, and dates back to 2010[^1].

For most of its history it was one of the most widely-depended-on packages in the .NET test ecosystem — hundreds of millions of NuGet downloads, a dependency of a large fraction of C# test suites. It is test-framework agnostic: it plugs into MSTest, NUnit, xUnit, MSpec, and NSpec by detecting which one is present and routing failures through its assertion exception type.

The defining fact about Fluent Assertions in 2026 is not technical, it is legal. **Version 8, released January 2025, changed the license from Apache-2.0 to a proprietary Xceed community license**[^2]. It remains free for open-source and non-commercial projects, but commercial use of v8+ now requires a paid license. Version 7 stays Apache-2.0 "indefinitely" per the maintainers[^3]. This split the community and spawned Apache-licensed forks. Any evaluation of this library today is really an evaluation of that decision.

## Getting Started

```bash
dotnet add package FluentAssertions
```

```csharp
using FluentAssertions;
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Adds_two_numbers()
    {
        int result = 2 + 2;

        result.Should().Be(4);
    }

    [Fact]
    public void Reports_helpful_collection_failure()
    {
        IEnumerable<int> numbers = new[] { 1, 2, 3 };

        numbers.Should().OnlyContain(n => n > 0);
        numbers.Should().HaveCount(3);
    }

    [Fact]
    public void Asserts_thrown_exceptions()
    {
        Action act = () => throw new ArgumentNullException("id");

        act.Should().Throw<ArgumentNullException>()
           .WithParameterName("id");
    }
}
```

Note the version pin: teams staying on the open-source line must lock to `7.*`. Installing `FluentAssertions` unpinned will pull v8+ and its license terms.

## Architecture / How It Works

The library is built entirely on C# **extension methods** hung off a `Should()` entry point. `Should()` returns a typed assertion object (`NumericAssertions<int>`, `StringAssertions`, `GenericCollectionAssertions<T>`, etc.) whose methods return `AndConstraint<T>` so calls chain with `.And`. There is no runtime magic — it is a large, hand-written surface of typed wrappers. The public API is locked down with approval tests (snapshot tests via the Verify library) so accidental breaking changes to method signatures are caught in CI[^4].

The flagship — and most complex — feature is **`Should().BeEquivalentTo()`**: structural object-graph comparison. Instead of reference or `Equals` comparison, it walks two object graphs by reflection, matching members by name and comparing values recursively, with extensive configuration (`options => options.Excluding(x => x.Id).WithStrictOrdering()`). This is what most teams actually adopt Fluent Assertions for. It handles cycles, collections, dictionaries, records, and anonymous types, and it drives the informative diff output.

Failure reporting flows through an `AssertionScope` and a caller-identification step that reads the calling source line (via the `CallerArgumentExpression` / stack inspection) to name the subject in the message — that is how it prints `numbers` rather than `value`. `using (new AssertionScope())` batches multiple assertions so all failures report together instead of stopping at the first.

The build targets a wide matrix — .NET Framework 4.7, .NET Core 2.1/3.0, .NET 6, .NET Standard 2.0 and 2.1 — which is why the codebase carries multi-targeting `#if` blocks and avoids newer BCL APIs in shared paths.

## Production Notes

**Licensing is now the primary operational concern.** Before adding or upgrading, confirm which line you are on. v7 (Apache-2.0) is a safe permanent choice for commercial code. v8+ requires a commercial license for commercial use; the community fork AwesomeAssertions is an Apache-2.0 continuation of the v7 API and is close to a drop-in replacement (change the `using` and package). Audit your transitive dependencies too — a library you consume may pull v8.

**`BeEquivalentTo` is a footgun at scale.** Because it is reflection-driven and config-heavy, it can silently pass when you expect it to fail (a member you forgot to compare) or fail cryptically on cycles, lazy-loaded EF navigation properties, or `DateTime`/`decimal` precision. Excluding members with lambdas is stringly-typed against refactors. It is also materially slower than direct equality; hot test suites with thousands of deep-graph comparisons feel it.

**Caller-name inference is fragile.** The nice "Expected numbers to…" subject naming depends on reading source expressions and can degrade to generic names under certain build configs, single-line chained calls, or trimming/AOT. It never affects correctness, only message quality.

**Async.** Use `await act.Should().ThrowAsync<T>()` for async delegates; mixing the sync `Throw` with async code is a common mistake that either fails to observe the exception or warns about un-awaited tasks.

**Analyzer.** The separate `FluentAssertions.Analyzers` package flags idiomatic-but-weaker patterns (e.g. `collection.Count.Should().Be(0)` → `collection.Should().BeEmpty()`) for better messages.

## When to Use / When Not

**Use when:**
- You want readable assertions and rich failure diffs across MSTest/NUnit/xUnit uniformly.
- You compare object graphs and want `BeEquivalentTo` rather than hand-writing per-field asserts.
- You are on the v7 Apache-2.0 line, or your project qualifies as non-commercial/open-source under the v8 terms.

**Avoid when:**
- You are commercial and unwilling to pay for v8 or pin to v7 forever — pick an Apache/MIT alternative instead.
- Your assertions are simple equality/throws checks — the framework-native `Assert` or a lighter library is enough.
- You need AOT/trimmed test hosts where caller-expression inference and heavy reflection are liabilities.

## Alternatives

- shouldly/shouldly — MIT, similar readable-assertion + good-message philosophy; the most common migration target after the v8 relicense.
- AwesomeAssertions/AwesomeAssertions — Apache-2.0 hard fork of Fluent Assertions v7; near drop-in for teams avoiding the commercial license.
- nunit/nunit — its built-in `Assert.That` constraint model already gives fluent-ish assertions with no extra dependency.
- xunit/xunit — `Assert.*` is terse and license-clean; pair with a snapshot library for structural comparison.
- VerifyTests/Verify — different shape (snapshot/approval testing); complements or replaces `BeEquivalentTo` for large object graphs.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2010–2011 | Initial releases by Dennis Doomen[^1]. |
| 4.0 | 2016 | Broad framework support, maturing object-graph comparison. |
| 5.0 | 2018 | .NET Standard targeting, API cleanup. |
| 6.0 | 2021 | Major release; `BeEquivalentTo` and diagnostics overhaul[^5]. |
| 7.0 | 2024 | Last Apache-2.0 major; remains open-source indefinitely[^3]. |
| 8.0 | 2025-01 | License changed to Xceed commercial/community terms[^2]. |

## References

[^1]: Fluent Assertions, "About". https://fluentassertions.com/about/
[^2]: Dennis Doomen, "Xceed and the future of Fluent Assertions" — Jan 2025. https://xceed.com/fluent-assertions-faq/
[^3]: Project README — "Version 7 will remain fully open-source indefinitely and receive bugfixes." https://github.com/fluentassertions/fluentassertions/blob/main/README.md
[^4]: Contribution docs on Approval.Tests / Verify-based public API snapshots. https://github.com/fluentassertions/fluentassertions/blob/main/CONTRIBUTING.md
[^5]: Fluent Assertions releases. https://github.com/fluentassertions/fluentassertions/releases

## Tags

csharp, dotnet, testing, assertions, unit-testing, tdd, bdd, test-framework, nuget, license-change, object-comparison
