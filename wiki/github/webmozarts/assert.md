# webmozarts/assert

> A single static `Assert` class of ~150 guard methods for validating method inputs and outputs, with consistent error messages and static-analyzer type narrowing.

[GitHub repo](https://github.com/webmozarts/assert) ·
[Packagist: webmozart/assert](https://packagist.org/packages/webmozart/assert) ·
[License: MIT](https://github.com/webmozarts/assert/blob/master/LICENSE)

## Overview

Webmozart Assert is a small PHP library by Bernhard Schussek (`@webmozart`, a long-time Symfony contributor) that provides a flat catalogue of assertion methods — `Assert::string()`, `Assert::greaterThan()`, `Assert::isInstanceOf()`, and so on. Each throws `Webmozart\Assert\InvalidArgumentException` (a subclass of SPL's `\InvalidArgumentException`) when the check fails. The intent is defensive programming at function boundaries: replace hand-written `if (!is_string($x)) throw ...` boilerplate with one expressive line.[^1]

The library exists because of a specific complaint about its ancestor, `beberlei/assert`: placeholder ordering in custom error messages was inconsistent across assertions, and that could not be fixed there without breaking backward compatibility. Webmozart Assert standardises this — `%s` is always the tested value, and `%2$s`, `%3$s`, … are assertion-specific extras (bounds, allowed values).[^1] That is the whole reason to prefer it over the older package, and it is a narrow reason; the two libraries are otherwise close cousins.

Its second, larger role today is as a static-analysis affordance. The assertion methods carry Psalm `@psalm-assert` annotations, so after `Assert::string($x)` a type checker knows `$x` is `string` for the rest of the scope. In modern PHP codebases this is often the primary value — runtime guards that double as type-narrowing hints for Psalm and PHPStan.[^2]

## Getting Started

```bash
composer require webmozart/assert
```

```php
use Webmozart\Assert\Assert;

class Employee
{
    public function __construct($id)
    {
        Assert::integer($id, 'The employee ID must be an integer. Got: %s');
        Assert::greaterThan($id, 0, 'The employee ID must be positive. Got: %s');
    }
}

new Employee('foobar');
// Webmozart\Assert\InvalidArgumentException:
//   The employee ID must be an integer. Got: string
```

## Architecture / How It Works

The library is essentially one class, `Assert`, holding static methods grouped by concern: type (`string`, `integer`, `isArray`, `isInstanceOf`), comparison (`eq`, `same`, `greaterThan`, `range`), string (`contains`, `regex`, `email`, `uuid`), file (`fileExists`, `readable`), object (`classExists`, `implementsInterface`), array (`keyExists`, `count`, `isList`, `isMap`), and function (`throws`, `isStatic`).

Two prefixes are not written out as methods; they are synthesised at runtime by `__callStatic`:

- `nullOr*` — runs the assertion only if the value is not `null` (`Assert::nullOrString($x)`).
- `all*` — applies the assertion to every element of an array or `\Traversable` (`Assert::allIsInstanceOf($items, Foo::class)`).

The two compose: `Assert::allNullOrString($list)`. Because these are magic, they don't exist as real symbols — IDEs and static analysers only understand them through bundled stubs and the Psalm/PHPStan plugins, not by reflection.

Failure flows through `reportInvalidArgument($message)`, which throws. A handful of `protected static` seams — `valueToString`, `typeToString`, `strlen` (which prefers `mb_strlen` when useful), `reportInvalidArgument`, and `__callStatic` — are documented override points, so subclasses can localise messages, represent value objects, or throw a domain-specific exception instead.[^1] The tight coupling worth understanding is not internal but external: the value of the library is realised through the static-analysis annotation layer, so upgrading PHP, Psalm, or PHPStan versions is entangled with the assertion stubs staying accurate.

## Production Notes

- **Assertions always run.** Unlike PHP's built-in `assert()` (which `zend.assertions` can strip in production), these are ordinary method calls that execute unconditionally. The checks are cheap, but in tight inner loops the call overhead is real; keep them at boundaries, not inside per-row hot paths.
- **These are programmer-error guards, not input validation.** They throw a single exception on the first failure with no way to collect multiple errors and no constraint metadata. For user-facing form/API validation where you need aggregated messages, reach for `symfony/validator` or `respect/validation` instead.
- **`InvalidArgumentException` extends the SPL class.** Broad `catch (\InvalidArgumentException)` blocks will swallow these. Many teams treat assertion failures as bugs that should crash, so avoid catching them generically.
- **Static-analysis narrowing needs the plugins.** PHPStan requires `phpstan/phpstan-webmozart-assert`; without it the analyser won't understand that `Assert::string()` narrows the type, and `all*`/`nullOr*` are especially opaque.[^3] Psalm ships assertion support natively and has an optional plugin for return-type inference (2.x).[^2]
- **Watch the loose type checks.** `integerish()` accepts numeric strings like `"123"`, `numeric()` accepts numeric strings, and `eq()`/`notEq()` use `==`. Pick `integer()`/`same()` when you mean strict.
- **2.0 raised the PHP floor and tightened typing.** The 2.x line added scalar/return type declarations and requires a modern PHP; subclasses that overrode the protected seams under 1.x may hit signature-compatibility errors on upgrade. Most callers of `Assert::` directly are unaffected.[^4]

## When to Use / When Not

**Use when:**
- You want terse, readable precondition checks at method boundaries in library or application code.
- You already run Psalm or PHPStan and want assertions that double as type-narrowing hints.
- You want consistent, customisable error-message placeholders (the reason to pick this over `beberlei/assert`).

**Avoid when:**
- You need to validate untrusted user input and report all errors at once — use a validation framework.
- You want a fluent, chainable assertion API (`Assert::that($v)->notEmpty()->string()`) — that is `beberlei/assert`'s model, not this one.
- You are on a very old PHP and cannot move to a 2.x-supported runtime.

## Alternatives

- beberlei/assert — the direct ancestor; use it when you want chained assertions and lazy assertion collections rather than a flat static catalogue.
- symfony/validator — use for user-facing validation with constraint metadata, groups, and aggregated error messages.
- respect/validation — use when you prefer a fluent, composable validator for input validation.
- azjezz/psl — use when you want a broader typed-by-contract standard library, not just guard assertions.
- phpstan/phpstan or vimeo/psalm — use when you want compile-time type verification instead of (or alongside) runtime guards.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2015 | Initial release; flat `Assert` class, consistent message placeholders vs beberlei/assert.[^1] |
| 1.x | 2015–2020 | Long-lived line; steady addition of assertions (array `isList`/`isMap`, string `uuid`/`ip`, Psalm annotations). |
| 2.0.0 | 2020 | Raised minimum PHP, added scalar/return type declarations, stricter internals.[^4] |
| 2.x | 2020–present | Maintenance and incremental assertions (e.g. lazy/callable message support, Psalm return-type plugin).[^2] |

## References

[^1]: Webmozart Assert README — FAQ, placeholder ordering, override seams. https://github.com/webmozarts/assert
[^2]: Psalm assertion syntax and the webmozart/assert return-type plugin. https://psalm.dev/docs/annotating_code/assertion_syntax/
[^3]: PHPStan Webmozart Assert extension. https://github.com/phpstan/phpstan-webmozart-assert
[^4]: Webmozart Assert releases on Packagist. https://packagist.org/packages/webmozart/assert

## Tags

php, assertions, validation, defensive-programming, preconditions, static-analysis, psalm, phpstan, composer-package, input-validation, library
