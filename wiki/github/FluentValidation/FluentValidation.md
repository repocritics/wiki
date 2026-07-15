# FluentValidation/FluentValidation

> A fluent-interface validation library for .NET that keeps validation rules out of your domain models and expresses them as strongly-typed lambda chains.

[GitHub repo](https://github.com/FluentValidation/FluentValidation) ·
[Official website](https://fluentvalidation.net) ·
[License: Apache-2.0](https://github.com/FluentValidation/FluentValidation/blob/main/License.txt)

## Overview

FluentValidation is a .NET library for building validation rules using a fluent
builder API and lambda expressions instead of validation attributes. Rules live
in a dedicated `AbstractValidator<T>` class rather than being scattered as
attributes across the model, which keeps DTOs and domain entities free of
validation concerns. It was created by Jeremy Skinner (originally on CodePlex
around 2008) and is now stewarded under the .NET Foundation[^1], though
development and support remain effectively one person's spare-time work — a fact
the README states plainly and backs with a sponsorship ask[^2].

The library's defining tension is its relationship with ASP.NET's automatic
validation. For years the headline use case was drop-in MVC/ASP.NET Core
integration where the framework validated incoming models for you. Skinner
deprecated that automatic integration in version 11 and removed the
`FluentValidation.AspNetCore` auto-validation path, arguing it was implicit,
hard to reason about, and encouraged putting validators in the request pipeline
where they silently ran[^3]. The recommended pattern is now explicit: inject
`IValidator<T>`, call `Validate`/`ValidateAsync` yourself, and decide what to do
with the result. This was a controversial break for teams who adopted
FluentValidation specifically for the magic, and it is the single most common
source of upgrade confusion.

## Getting Started

```
dotnet add package FluentValidation
```

```csharp
using FluentValidation;

public class CustomerValidator : AbstractValidator<Customer> {
  public CustomerValidator() {
    RuleFor(x => x.Surname).NotEmpty();
    RuleFor(x => x.Forename).NotEmpty().WithMessage("Please specify a first name");
    RuleFor(x => x.Discount).NotEqual(0).When(x => x.HasDiscount);
    RuleFor(x => x.Address).Length(20, 250);
    RuleFor(x => x.Email).EmailAddress();
  }
}

var validator = new CustomerValidator();
ValidationResult result = validator.Validate(new Customer());

if (!result.IsValid) {
  foreach (var error in result.Errors)
    Console.WriteLine($"{error.PropertyName}: {error.ErrorMessage}");
}
```

For DI-based apps, register validators once and resolve `IValidator<T>`:

```csharp
// Startup / Program.cs
builder.Services.AddValidatorsFromAssemblyContaining<CustomerValidator>();
```

## Architecture / How It Works

A validator is a class deriving from `AbstractValidator<T>`. In the constructor
you call `RuleFor(x => x.Property)` to open a rule chain; each chained method
(`NotEmpty`, `Length`, `Must`, `EmailAddress`, …) appends a `PropertyValidator`
to that rule. At validation time the library walks the rules, evaluates the
compiled member-access expression against the instance, runs each property
validator, and collects `ValidationFailure` objects into a `ValidationResult`.
The lambda in `RuleFor` is used both to read the value and to derive the default
property name for error messages.

Key building blocks worth knowing:

- **`CascadeMode`** — controls whether a rule chain stops at the first failure
  or evaluates every validator. This can be set per-rule, per-validator, or
  globally, and the semantics of the global vs. rule-level cascade changed across
  major versions, which is a frequent source of "why is it still running the
  next check" confusion.
- **`Must` / `MustAsync`** — the escape hatch to arbitrary predicate logic when
  no built-in validator fits. `MustAsync` plus `ValidateAsync` enable
  database/remote checks (e.g. uniqueness), but async rules require the async
  entry point — calling synchronous `Validate` on a validator containing async
  rules throws.
- **`SetValidator` / `RuleForEach`** — compose child validators for nested
  objects and collections. Deep object graphs become trees of validators.
- **`When` / `Unless` / `WithMessage` / `WithName`** — conditional and message
  customization applied to preceding rules; ordering and grouping semantics
  matter and are a common footgun.
- **Localization** — error messages resolve through a replaceable
  `ILanguageManager`, with built-in translations for many languages.

Validators are intended to be stateless and safe to register as singletons; all
per-request state lives in the `ValidationContext` passed at call time.

## Production Notes

- **Automatic ASP.NET validation is gone.** If you are on version 11+ do not
  expect model binding to validate for you. Wire validation explicitly (manual
  `ValidateAsync`, a MediatR pipeline behavior, or a minimal-API filter). Old
  tutorials showing `AddFluentValidation()` auto-validation describe a removed
  code path[^3].
- **Don't put stateful data in validator fields.** Because validators are
  typically registered as singletons via `AddValidatorsFromAssembly`, mutable
  instance fields are shared across requests and cause race conditions. Pass
  context through `ValidationContext` / root-context data instead.
- **Async vs sync mismatch throws.** A validator containing `MustAsync`,
  `WhenAsync`, or `CustomAsync` rules must be invoked with `ValidateAsync`.
  Calling `Validate` on it raises an `AsyncValidatorInvokedSynchronouslyException`.
- **`Must` predicates run on every validation.** Expensive predicates (DB hits,
  regex over large input) are re-evaluated each call; guard them with `When` and
  prefer `Cascade(Stop)` so a cheap `NotEmpty` short-circuits before an
  expensive check runs.
- **Package split matters.** The core `FluentValidation` package has no ASP.NET
  dependency; DI helpers live in `FluentValidation.DependencyInjectionExtensions`.
  Pinning the wrong companion package version against a new core release is a
  common build break.
- **It validates, it does not sanitize or transform.** FluentValidation reports
  failures; it will not coerce, trim, or normalize values for you. Treat it as a
  gate, not a mapper.

## When to Use / When Not

**Use when:**
- Validation rules are non-trivial, conditional, or cross-field and would be
  awkward as attributes.
- You want validation logic testable in isolation, separate from models.
- You need async rules (uniqueness, remote lookups) as first-class citizens.
- You are building a CQRS/MediatR pipeline and want a validation behavior stage.

**Avoid when:**
- Rules are simple and declarative — built-in DataAnnotations attributes ship
  with .NET and need no dependency.
- You specifically want framework-driven automatic model validation with zero
  glue code; that path was removed in v11.
- You need input sanitization/transformation, not just pass/fail reporting.

## Alternatives

- dotnet/runtime (`System.ComponentModel.DataAnnotations`) — use the built-in
  attribute validators when rules are simple, declarative, and you want zero
  extra dependencies.
- DamianEdwards/MiniValidation — use for minimal-API apps that want lightweight
  DataAnnotations execution without a separate rule-class DSL.
- ardalis/GuardClauses — use for defensive argument/precondition guards inside
  methods, which is a different scope than model validation.
- MediatR (jbogard/MediatR) — not a validator itself, but the pipeline it
  provides is the usual host for FluentValidation as a request-validation stage.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2008 | Originated on CodePlex, authored by Jeremy Skinner[^1]. |
| 8.0 | 2018 | Broad .NET Standard 2.0 targeting; ASP.NET Core integration. |
| 9.0 | 2020 | API cleanup, removal of long-deprecated members. |
| 10.0 | 2021 | Further modernization; DI extension refinements[^4]. |
| 11.0 | 2022 | Automatic ASP.NET validation deprecated/removed; explicit validation recommended[^3]. |

## References

[^1]: FluentValidation is part of the .NET Foundation; original author Jeremy Skinner. https://dotnetfoundation.org/
[^2]: FluentValidation README — sponsorship note ("developed and supported by @JeremySkinner for free in his spare time"). https://github.com/FluentValidation/FluentValidation
[^3]: FluentValidation docs, "ASP.NET Core" — deprecation of automatic validation and the recommended manual approach. https://docs.fluentvalidation.net/en/latest/aspnet.html
[^4]: FluentValidation documentation (versioned upgrade guides). https://docs.fluentvalidation.net/

## Tags

csharp, dotnet, validation, aspnet-core, fluent-interface, lambda-expressions, dependency-injection, library, dotnet-foundation, backend
