# nsubstitute/NSubstitute

> A .NET mocking library that hides test-double configuration behind extension methods and thread-local state — succinct to read, easy to misuse.

[GitHub repo](https://github.com/nsubstitute/NSubstitute) ·
[Official website](https://nsubstitute.github.io) ·
[License: BSD-3-Clause](https://github.com/nsubstitute/NSubstitute/blob/main/LICENSE.txt)

## Overview

NSubstitute is a mocking (test-double) library for .NET, distributed on NuGet, whose stated goal is a "friendly substitute for .NET mocking libraries"[^1]. Rather than the arrange/verify vocabulary of classic frameworks, it collapses stub, mock, spy, and fake into a single concept: `Substitute.For<T>()` returns an object you configure with fluent extension methods like `.Returns(...)` and assert against with `.Received()`. The design goal is that a test reads close to natural language, with minimal lambdas and setup noise.

The defining tension is that this readability is bought with implicit static state. `.Returns()` and `Arg.Any<T>()` are not called *on* the substitute in an object-oriented sense — they record intent into an ambient `SubstitutionContext` that correlates the *last* intercepted call with the configuration that follows it[^2]. That is what makes `_calc.Add(1, 2).Returns(3)` legal C#, but it is also why the same expressiveness produces confusing failures when calls are non-virtual, when argument matchers are used outside a substituted call, or when tests run in parallel. NSubstitute even ships a companion Roslyn analyzer package precisely because a portion of correct usage cannot be enforced by the type system[^3].

The repo dates to 2009 and is mature and still actively maintained (last push 2026-07-12), with roughly 3.0k stars and 280 forks. It sits in a three-way ecosystem with Moq and FakeItEasy; NSubstitute is generally chosen for syntax preference rather than a unique capability.

## Getting Started

```bash
dotnet add package NSubstitute
# recommended: compile-time misuse detection
dotnet add package NSubstitute.Analyzers.CSharp
```

```csharp
using NSubstitute;
using NUnit.Framework;

public interface ICalculator { int Add(int a, int b); string Mode { get; set; } }

[Test]
public void Substitute_returns_and_verifies()
{
    var calc = Substitute.For<ICalculator>();

    calc.Add(1, 2).Returns(3);                 // stub a return value
    calc.Add(Arg.Any<int>(), Arg.Any<int>())   // matcher + computed return
        .Returns(ci => (int)ci[0] + (int)ci[1]);

    Assert.That(calc.Add(1, 2), Is.EqualTo(3));

    calc.Received().Add(1, 2);                  // verify a call happened
    calc.DidNotReceive().Add(5, 7);
}
```

## Architecture / How It Works

NSubstitute does not generate proxies itself — it builds on **Castle DynamicProxy** (from castleproject/Core) to emit a runtime subclass of the requested type whose members route through an interceptor[^4]. This is the root of most of its capabilities *and* limitations: DynamicProxy can only override what the CLR lets it override — interface members and `virtual` members of non-sealed classes. Non-virtual methods, sealed classes, `static` members, and extension methods are invisible to the proxy and silently call the real implementation instead of the substitute.

The fluent surface is a set of **extension methods over a thread-scoped context**. When you invoke `calc.Add(1, 2)` inside a test, the interceptor records that call as "the last call" in the ambient `SubstitutionContext`. The subsequent `.Returns(3)` — an extension method on the returned `int` — reaches back into that context and attaches the configuration to the recorded call. `Received()` works the same way: it flips the context into a "check the next call" mode. This is why ordering matters and why the API looks like it is mutating values that are actually just carriers.

**Argument matchers** (`Arg.Any<T>()`, `Arg.Is<T>(...)`) are the sharpest edge. Each matcher call pushes an entry onto a per-context queue and returns a `default(T)` placeholder; the interceptor dequeues them when the enclosing call is recorded. The rule is all-or-nothing: within one call you may not mix a raw value and a matcher for reference/nullable arguments without using `Arg.Is`. A matcher evaluated outside a substituted call (e.g. stored in a variable, or used on a non-virtual method) leaks onto the queue and corrupts the *next* unrelated call — a failure that surfaces far from its cause. The analyzer package exists mainly to catch these statically.

Other internals worth knowing: `ForPartsOf<T>()` creates a partial substitute that calls real code unless overridden (and still invokes the real constructor); `Received(n)`, `ReceivedWithAnyArgs()`, and `Received.InOrder(...)` cover call verification; `Returns` accepts multiple values to script a sequence; and `Raise.Event<T>()` fires events. The `SubstitutionContext` is held in an `AsyncLocal`, which is what makes async/await flows and (mostly) parallel tests behave — but see Production Notes.

## Production Notes

- **The single biggest footgun is silent no-ops on non-virtual members.** `Substitute.For<AConcreteClass>()` compiles and runs, but any `.Returns()` on a non-virtual method does nothing and the real method executes. Prefer substituting interfaces; if you must substitute a class, make members `virtual` or you will chase phantom behavior. Install the analyzer — it is the only thing that turns these into build errors.
- **Matcher-queue corruption crosses test boundaries.** A leaked `Arg` matcher from one assertion can fail a *later* test in the same context. Symptoms are non-local and order-dependent. Keep matchers strictly inside the call they belong to; never hoist `Arg.Any<T>()` into a local.
- **Parallelism.** Because configuration flows through an `AsyncLocal` context, tests that share substitutes across threads, or frameworks that reuse execution contexts, can interleave configuration. The safe pattern is one substitute set constructed and asserted within a single test's async flow; avoid static/shared substitutes under xUnit's parallel collections.
- **`ForPartsOf` runs real constructors** and real code by default. Combined with `.Returns()` (which *calls* the member before overriding it), partial substitutes can trigger real side effects during setup; use `.Configure().<Member>().Returns(...)` / `When(...).DoNotCallBase()` to avoid the first real invocation.
- **`out`/`ref` and generics** are supported but verbose, and matcher support for them is easy to get subtly wrong; verify with an explicit test the first time.
- **Upgrade pain: 5.0 (2023) dropped legacy target frameworks.** Projects on old .NET Framework/`netstandard1.x` cannot move to 5.x without retargeting[^5]. This is the main version-boundary teams hit.

## When to Use / When Not

**Use when:**
- You want the least-ceremony mocking syntax in a codebase that already tests against interfaces.
- Your team values readable assertions (`Received()`) over an explicit arrange/verify vocabulary.
- You can adopt `NSubstitute.Analyzers` to backstop the runtime-magic API.

**Avoid when:**
- You need to mock non-virtual methods, sealed classes, or `static` members — no DynamicProxy-based library can do this; reach for a commercial IL-rewriting tool instead.
- You want misuse to be a compile error out of the box; the core library's power comes from patterns the type system cannot police.
- Your design leans on concrete-class substitution with real constructors you cannot safely run.

## Alternatives

- devlooped/moq — the original arrange-act-assert .NET mock and still the most widely used; pick it for ecosystem size and `Setup`/`Verify` explicitness (note the 2023 SponsorLink episode that soured some users).
- FakeItEasy/FakeItEasy — same "one kind of fake, friendly syntax" philosophy as NSubstitute; choose it if you prefer its API shape but want the same DynamicProxy capabilities/limits.
- castleproject/Core — not a mocking library but the DynamicProxy engine all of the above sit on; use directly when you need raw interception without a mocking DSL.
- tonerdo/pose — shims/replaces static and non-virtual calls at runtime; use it for the exact cases DynamicProxy-based libraries cannot cover, without a commercial license.
- Typemock Isolator / Telerik JustMock (commercial) — IL-rewriting profilers that can fake static, sealed, and non-virtual members; use when legacy code is untestable by any proxy-based tool.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2009-10-23 | Project started on GitHub[^6]. |
| 1.x | ~2010–2015 | Established the `Substitute.For` / `Returns` / `Received` API. |
| 2.0 | 2016 | Modernized Castle DynamicProxy dependency; framework support changes. |
| 3.0 | ~2017 | `netstandard` support for .NET Core era. |
| 4.0 | ~2018 | Analyzer ecosystem (`NSubstitute.Analyzers`) matured alongside. |
| 5.0 | 2023 | Dropped legacy target frameworks; minimum retargeting required[^5]. |

## References

[^1]: NSubstitute README — "designed as a friendly substitute for .NET mocking libraries." https://github.com/nsubstitute/NSubstitute
[^2]: NSubstitute docs, "How NSubstitute works" — argument matching and the substitution context. https://nsubstitute.github.io/help/how-nsub-works/
[^3]: NSubstitute.Analyzers — Roslyn analyzers for detecting incorrect usage. https://github.com/nsubstitute/NSubstitute.Analyzers
[^4]: Castle DynamicProxy (castleproject/Core), the proxy engine NSubstitute builds on. https://github.com/castleproject/Core
[^5]: NSubstitute releases and changelog (5.0 target-framework changes). https://github.com/nsubstitute/NSubstitute/releases
[^6]: GitHub repository metadata, `created_at` 2009-10-23. https://github.com/nsubstitute/NSubstitute

## Tags

csharp, dotnet, mocking, test-doubles, unit-testing, testing, nuget, castle-dynamicproxy, roslyn-analyzers, mock, stub
