# mockery/mockery

> A PHP mock object framework with a fluent, DSL-style API for defining test doubles in unit tests.

[GitHub repo](https://github.com/mockery/mockery) ·
[Official website](http://docs.mockery.io/en/stable/) ·
[License: BSD-3-Clause](https://github.com/mockery/mockery/blob/1.6.x/LICENSE)

## Overview

Mockery is a standalone test-double framework for PHP, first published in 2009 and
still actively maintained — the default branch is `1.6.x` and the last push was in
May 2026. It exists to make mocks, stubs, and spies readable: instead of PHPUnit's
builder chains, you write `$db->shouldReceive('find')->with(123)->once()->andReturn($row)`,
which reads close to a sentence describing the interaction under test[^1]. Roughly
10.7k stars and 466 forks make it the most widely used third-party mocking library
in the PHP ecosystem, and it ships as a `require-dev` dependency in a large share of
Composer projects.

The framework's defining choice is that it is a *separate* library from your test
runner. It integrates with PHPUnit (via `MockeryTestCase` or the
`MockeryPHPUnitIntegration` trait) but does not depend on it, and can run under
PHPSpec or any other harness[^1]. That independence is also its central tension:
Mockery maintains its own verification lifecycle that PHPUnit does not know about.
If you never call `Mockery::close()`, expectations are silently not asserted and
mock state leaks between tests — the single most common Mockery footgun.

The project was formerly hosted at `padraic/mockery` and transferred to the
`mockery/mockery` organization; old URLs still redirect, but remotes should be
updated[^2].

## Getting Started

```sh
composer require --dev mockery/mockery
```

```php
use Mockery;

class Book {}
interface BookRepository {
    public function find(int $id): Book;
}

// Create a double satisfying the type hint
$repo = Mockery::mock(BookRepository::class);

// Stub: canned return, call count irrelevant
$repo->allows()->find(123)->andReturns(new Book());

// Expectation: must be called exactly once
$repo->expects()->find(123)->andReturns(new Book());

// ... exercise the system under test ...

Mockery::close(); // verifies expectations; REQUIRED
```

Under PHPUnit, extend `Mockery\Adapter\Phpunit\MockeryTestCase` or `use` the
`MockeryPHPUnitIntegration` trait so that `Mockery::close()` runs automatically in
`tearDown()`[^1].

## Architecture / How It Works

Mockery builds mock classes at runtime through **code generation**. When you call
`Mockery::mock(SomeInterface::class)`, it reflects over the target type, assembles a
PHP source string for a class that implements the interface (or extends the class),
and evaluates it into a real, uniquely named class in memory. The returned object is
an instance of that generated class, with each method routed through Mockery's
expectation-matching engine[^3].

Every double is registered in a global **container** (`Mockery::getContainer()`).
Calls are recorded against expectations defined by argument matchers (`with(...)`,
`Mockery::on(...)`, `Mockery::type(...)`, `Mockery::any()`), ordering constraints
(`ordered()`, `globally()`), and count constraints (`once()`, `times(n)`,
`atLeast()`). `Mockery::close()` walks the container, asserts that every `expects()`
was satisfied, and tears the container down. This is why `close()` is load-bearing:
it *is* the verification step, not cleanup.

Three double flavors sit on the same machinery:

- **Regular mocks** reject any call not explicitly allowed or expected.
- **Spies** (`Mockery::spy()`, i.e. `shouldIgnoreMissing()`) accept everything and
  return null, then verify after the fact with `shouldHaveReceived()`[^1].
- **Partial mocks** wrap a real instance and override only selected methods.

Two special modes bend PHP's type system. **Alias mocks** (`Mockery::mock('alias:Foo')`)
define a class in the global namespace to intercept static calls, and **overload
mocks** (`Mockery::mock('overload:Foo')`) stand in for `new` instantiation of a
not-yet-loaded class. Both manipulate class loading and are the source of most of
Mockery's sharp edges.

## Production Notes

- **`Mockery::close()` is mandatory and easy to forget.** Without it, `expects()`
  assertions never run — tests go green while verifying nothing — and mock state
  survives into the next test. Always use the PHPUnit trait/base class rather than
  hand-managing it.
- **Alias and overload mocks pollute global state.** Because they define real classes
  into the process, any PHPUnit test using them must run with
  `@runInSeparateProcess` and `@preserveGlobalState disabled`, or a later test that
  autoloads the real class will fatal with a redeclaration error[^3]. This is slow
  and a frequent CI flake source.
- **Final classes/methods cannot be mocked directly.** Mockery generates subclasses,
  so `final` blocks it. The common workaround is `dg/bypass-finals`, which strips the
  keyword at load time — powerful and dangerous.
- **PHP version deprecations.** Because doubles are dynamically generated, Mockery has
  historically tripped PHP's evolving strictness (notably the PHP 8.2 dynamic-property
  deprecation). Keep Mockery current with your PHP runtime; older lines emit notices
  on newer PHP.
- **Semantic versioning is qualified.** The maintainers explicitly reserve the right
  to change internals even when a class is non-`final` or a method is public — only
  the documented API is guaranteed[^1]. Do not build tooling against Mockery internals.
- **Argument matching by reference and object identity** can surprise: `with($obj)`
  matches by `==` (loose equality) unless you use `Mockery::mustBe($obj)` for `===`.

## When to Use / When Not

**Use when:**
- You want expressive, readable expectations across many interactions and value the
  `shouldReceive` / `allows` / `expects` DSL over PHPUnit's mock builder.
- You need spies, partial mocks, ordered expectations, or trait mocking that the
  built-in PHPUnit mocker handles awkwardly or not at all.
- You are testing against interfaces and abstract dependencies (its happy path).

**Avoid when:**
- You want zero extra dependencies and PHPUnit's `createMock()` already covers your
  needs — a simple stub does not justify a second framework.
- Your design leans on mocking `final`/`static`/global code — that signals coupling
  Mockery can only paper over with fragile alias/overload tricks.
- You need to mock built-in PHP functions (e.g. `time()`, `fopen()`) — that is
  `php-mock/php-mock` territory, not Mockery.

## Alternatives

- sebastianbergmann/phpunit — its built-in `createMock()` / `MockBuilder` needs no
  extra dependency; use it when your mocking needs are simple and PHPUnit-only.
- phpspec/prophecy — promise/prediction-based doubles; use it if you prefer its
  opinionated object-oriented style over Mockery's method-chain DSL.
- php-mock/php-mock — use it specifically to mock native PHP functions in a
  namespace, which Mockery cannot do.
- dg/bypass-finals — not a mock library but a companion; use it alongside Mockery
  to mock `final` classes.
- friendsofphp/phpstan-mocks / AspectMock — AOP-based static/final mocking; largely
  unmaintained, consider only for legacy suites already depending on it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2010–2017 | Original `padraic/mockery`; the README served as documentation. |
| 1.0 | 2018 | Introduced the `allows()` / `expects()` DSL alongside `shouldReceive()`[^1]. |
| 1.3 | 2019 | Last line to support PHP 5 and HHVM 3[^1]. |
| 1.4 | 2020 | Dropped PHP 5 / HHVM; PHP 7.3+ baseline. |
| 1.5 | 2022 | PHP 8.1 support. |
| 1.6 | 2023– | Current line; PHP 8.2+ dynamic-property fixes and continued runtime support. |

## References

[^1]: Mockery README and documentation. https://github.com/mockery/mockery and http://docs.mockery.io/en/stable/
[^2]: Mockery README, "A new home for Mockery" — repository transferred from `padraic/mockery` to `mockery/mockery`. https://github.com/mockery/mockery#a-new-home-for-mockery
[^3]: Mockery documentation, "Creating test doubles" and "Mocking public static methods / hard dependencies". http://docs.mockery.io/en/latest/reference/creating_test_doubles.html

## Tags

php, testing, mocking, test-doubles, unit-testing, phpunit, mock, stub, spy, dsl, dev-tools
