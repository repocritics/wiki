# devlooped/moq

> The most widely used mocking library for .NET — and the one whose 2023 SponsorLink incident became a case study in supply-chain trust.

[GitHub repo](https://github.com/devlooped/moq) ·
[NuGet](https://www.nuget.org/packages/Moq) ·
[License: BSD-3-Clause](https://github.com/devlooped/moq/blob/main/License.txt)

## Overview

Moq (pronounced "Mock-you") is a mocking library for .NET that lets tests
supply fake implementations of interfaces and overridable class members, set
up their return values and behavior, and later verify how they were called. It
was the first .NET mocking library built around LINQ expression trees and
lambdas rather than the record/replay idiom that dominated earlier tools like
RhinoMocks[^1]. `mock.Setup(x => x.Method(arg)).Returns(value)` reads as
ordinary C#, gets full IntelliSense and refactoring support, and needs almost
no mocking vocabulary to use — which is most of why it became the default
choice across the ecosystem.

The project was created around 2007 by Daniel Cazzulino ("kzu") and others at
Clarius, Manas, and InSTEDD, moved to GitHub in 2012, and is now maintained
under the devlooped organization[^2]. It remains the most-downloaded mocking
package on NuGet by a wide margin. The defining tension for Moq in 2026 is not
technical but reputational. In
August 2023 a point release quietly bundled a closed-source dependency,
SponsorLink, that read and hashed the local git user's email at build time and
phoned home[^3]. The backlash was severe enough that many teams migrated to
NSubstitute or FakeItEasy on principle, and the episode is now the first thing
experienced .NET developers mention about the library. The code was reverted
within days, but the trust cost persists.

## Getting Started

```bash
dotnet add package Moq
```

```csharp
using Moq;
using Xunit;

public interface IEmailGateway {
    bool Send(string to, string body);
}

public class NotifierTests {
    [Fact]
    public void Notify_SendsEmail_Once() {
        var gateway = new Mock<IEmailGateway>();
        gateway.Setup(g => g.Send(It.IsAny<string>(), "hello"))
               .Returns(true);

        var notifier = new Notifier(gateway.Object);
        notifier.Notify("hello");

        gateway.Verify(g => g.Send("user@example.com", "hello"), Times.Once());
    }
}
```

`Mock.Of<T>` ("Linq to Mocks") builds a configured instance from a single
predicate, useful when you only need state, not verification:

```csharp
IEmailGateway gateway = Mock.Of<IEmailGateway>(g => g.Send("a", "b") == true);
```

## Architecture / How It Works

Moq does not generate code from strings or interfaces of its own. It builds on
Castle DynamicProxy[^4], which at runtime emits a proxy subclass that overrides
the target's virtual members. `new Mock<IFoo>().Object` returns an instance of
that generated proxy. Every call passes through an interceptor that consults the
setups recorded on the mock.

Setups are LINQ expression trees, not delegates. `Setup(x => x.M(It.IsAny<int>()))`
is never executed as code — Moq walks the expression to identify the target
member and to extract argument matchers (`It.IsAny`, `It.Is`, literals). This is
what gives the API its type safety and refactoring resilience, and also what
constrains it: the expression must terminate in an overridable member.

That constraint is the single most important thing to understand about Moq. Because
interception happens through a proxy subclass, Moq can only intercept:

- interface members,
- `virtual` and `abstract` methods and properties on classes,
- `protected` overridable members (via the string-based `Protected()` API).

It **cannot** intercept static methods, non-virtual instance methods, sealed
classes, `private` members, or extension methods. Attempting to `Setup` a
non-virtual member throws `NotSupportedException` at setup time. Tools that mock
those (TypeMock Isolator, Telerik JustMock) work by attaching a CLR profiler and
rewriting the JIT, a fundamentally different and heavier mechanism.

Two behavior modes govern unconfigured calls. `MockBehavior.Loose` (the default)
returns default values for anything not set up; `MockBehavior.Strict` throws on
any call that was not explicitly configured. `DefaultValue.Mock` extends loose
behavior to auto-create recursive mocks for reference-type returns. Verification
is a separate phase: the interceptor records every invocation, and `Verify` /
`VerifyAll` replay those records against expected call counts (`Times`).

## Production Notes

**The SponsorLink incident (4.20.0, August 2023).** A minor release added
SponsorLink as a transitive dependency. It ran a build-time analyzer that
extracted the git config email, hashed it, and sent it to a remote service to
check sponsorship status[^3]. Because Moq sits in the test dependency tree of an
enormous number of projects, this shipped silently into countless CI pipelines
and raised GDPR and supply-chain concerns. It was removed in 4.20.2 within days.
Practical fallout that still matters: pin your Moq version, avoid the 4.20.0 and
4.20.1 builds specifically, and audit analyzer/build dependencies rather than
assuming a mocking library is inert.

**Maintenance cadence.** The library is mature and effectively feature-complete;
the latest release is v4.20.72 (September 2024)[^5]. The repository still sees
commits, but releases are infrequent and development is centered on a single
primary maintainer. Read this as stability, not abandonment — but do not expect
rapid response to feature requests.

**Strict mocks are brittle.** `MockBehavior.Strict` couples the test to the exact
set of calls the implementation makes. Any incidental call — a logging hook, an
extra property read — fails the test. Most teams standardize on loose mocks and
verify only the interactions they actually care about.

**Over-mocking.** Because setting up a mock is so cheap, Moq makes it easy to
write tests that assert against implementation detail (which methods were called
in which order) rather than observable behavior. These tests break on every
refactor. This is a usage discipline problem, not a library defect, but it is the
most common way Moq test suites become a maintenance liability.

**No CLR-level power.** If your code under test calls `DateTime.Now`, a static
factory, or a sealed third-party class directly, Moq cannot help — the fix is a
seam (an injected interface), not a Moq feature. Async members use `ReturnsAsync`
/ `ThrowsAsync`; `out`/`ref` and events are supported but have their own syntax.

## When to Use / When Not

**Use when:**
- You are unit-testing .NET code that already depends on interfaces or abstract
  types (idiomatic DI-based design).
- You want the lowest learning curve and the most Stack Overflow / LLM coverage
  of any .NET mocking tool.
- You need state-based setup and interaction verification in one library.

**Avoid / look elsewhere when:**
- The SponsorLink history is a dealbreaker for your org's supply-chain policy —
  NSubstitute and FakeItEasy are clean alternatives with similar ergonomics.
- You must mock static, sealed, or non-virtual members without changing the
  design — you need a profiler-based tool (JustMock, TypeMock) instead.
- You want a syntax with no `.Object` and no `Setup`/`Verify` ceremony —
  NSubstitute's substitute-is-the-object model is lighter.

## Alternatives

- nsubstitute/NSubstitute — friendlier syntax, no `.Object` indirection; the most
  common destination for teams leaving Moq after 2023.
- FakeItEasy/FakeItEasy — single `A.Fake<T>()` API, similar capability surface;
  another clean-dependency choice.
- Use Telerik JustMock or TypeMock Isolator when you must mock static, sealed, or
  non-virtual members without introducing seams (commercial, profiler-based).
- Use Microsoft Fakes (Shims) when you need to substitute framework/static calls
  and are on Visual Studio Enterprise.
- JasonBock/Rocks — source-generator-based mocking, no runtime proxy; use when you
  want compile-time-generated mocks and AOT friendliness.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | ~2007 | Created by kzu et al. at Clarius/Manas/InSTEDD; first LINQ-based .NET mocking lib[^2]. |
| — | 2012-03 | Repository moved to GitHub. |
| 4.x | ongoing | Long-lived stable line; Castle DynamicProxy interception, Linq to Mocks. |
| 4.20.0 | 2023-08 | Bundled SponsorLink; extracted hashed git email at build time — major backlash[^3]. |
| 4.20.2 | 2023-08 | SponsorLink removed. |
| 4.20.72 | 2024-09-07 | Latest release as of this writing[^5]. |

## References

[^1]: Moq README — "the only mocking library ... to take full advantage of .NET Linq expression trees," contrasting the record/replay model. https://github.com/devlooped/moq
[^2]: devlooped/moq repository, created 2012-03-11 on GitHub; original copyright 2007, Clarius/Manas/InSTEDD. https://github.com/devlooped/moq
[^3]: Reporting and issue thread on Moq 4.20.0 bundling SponsorLink and extracting the git user email at build time (August 2023). https://github.com/devlooped/moq/issues/1372
[^4]: Castle DynamicProxy — the runtime proxy-generation library Moq uses for interception. https://github.com/castleproject/Core
[^5]: Latest release v4.20.72, published 2024-09-07. https://github.com/devlooped/moq/releases/latest

## Tags

dotnet, csharp, testing, mocking, unit-testing, test-doubles, castle-dynamicproxy, nuget, supply-chain, tdd
