# thecodingmachine/safe

> Core PHP functions rewritten to throw exceptions instead of returning `false`, plus a PHPStan rule that flags the unsafe originals.

[GitHub repo](https://github.com/thecodingmachine/safe) ·
[Packagist](https://packagist.org/packages/thecodingmachine/safe) ·
[License: MIT](https://github.com/thecodingmachine/safe/blob/master/LICENSE)

## Overview

Safe-PHP addresses a design wart that predates exceptions in the language: most
of PHP's ~1000 built-in functions signal failure by returning `false` (or `null`,
or `-1`) rather than throwing. Code that ignores those sentinels — which is most
code — silently propagates bad values. `file_get_contents()` returning `false`,
then fed to `json_decode()`, is the canonical example[^1].

Safe redeclares each fallible core function under the `Safe\` namespace with an
identical signature, checks the return, and throws a typed exception on failure.
You opt in per-call with `use function Safe\file_get_contents;`. The functions are
not hand-written: they are code-generated from the official PHP documentation's
XML sources, which is how the library keeps ~1000 wrappers in sync with the
language[^2].

The defining tension is ergonomics versus explicitness. Safe only helps on files
where you remembered to import the `Safe\` variant, so its real value comes bundled
with the companion PHPStan rule that fails the build whenever the unsafe original
is used[^3]. Without static analysis wired in, Safe is easy to forget and provides
little guarantee; with it, the pairing approximates a checked-exceptions discipline
that PHP itself lacks.

## Getting Started

```bash
composer require thecodingmachine/safe
# highly recommended — the enforcement half:
composer require --dev thecodingmachine/phpstan-safe-rule
```

```php
use function Safe\file_get_contents;
use function Safe\json_decode;

// Throws Safe\Exceptions\FilesystemException or JsonException on failure,
// instead of returning false and corrupting downstream state.
$content = file_get_contents('foobar.json');
$foobar  = json_decode($content, true);
```

Enable the PHPStan rule so plain `file_get_contents()` becomes a build error:

```yaml
# phpstan.neon
includes:
    - vendor/thecodingmachine/phpstan-safe-rule/phpstan-safe-rule.neon
```

## Architecture / How It Works

The wrappers are generated, not maintained by hand. A build script parses the PHP
documentation (the docbook XML that also produces php.net), extracts each function's
signature and its documented failure return, and emits a `Safe\<name>()` function
that calls the global original and throws when it sees the failure sentinel[^2].
The generated functions live in ~85 files grouped by extension (filesystem, JSON,
strings, curl, GD, etc.), each registered through Composer's `files` autoload key.

Because Composer `files` autoloading is eager, every one of those ~85 files loads
on every request — Safe functions cannot be tree-shaken or lazily loaded. This is
the source of the library's fixed per-request cost (see Production Notes).

Exceptions are typed per module. Each throws a class such as
`Safe\Exceptions\FilesystemException`, `Safe\Exceptions\JsonException`, or
`Safe\Exceptions\CurlException`; all implement a common `SafeExceptionInterface`
and extend PHP's `\ErrorException`, so you can catch narrowly by module or broadly
by the shared base[^4]. Safe also ships `Safe\DateTime` and `Safe\DateTimeImmutable`
whose methods throw instead of returning `false`.

For migrating existing code, Safe bundles a Rector configuration
(`rector-migrate.php`) that rewrites bare calls to their `Safe\` equivalents in
bulk[^5]. The set of wrapped functions is not fixed across major versions: v2
(PHP 8.0+) dropped a number of wrappers for functions that PHP 8 itself changed to
throw `TypeError`/`ValueError` natively, since re-wrapping them added no value.

## Production Notes

- **The Rector migration is a "dumb" replacement.** It swaps the namespace but does
  not touch existing error handling. Code like `if (!mkdir($p)) { ... }` becomes
  `if (!\Safe\mkdir($p)) { ... }` — but `Safe\mkdir` now *throws* on failure, so the
  `false` branch is dead and the error escapes as an uncaught exception. You must
  manually convert those sites to `try/catch`[^5]. Audit every touched conditional
  after running it.
- **Upgrading Safe can break working code.** Because v2 removed wrappers for functions
  PHP 8 made throw natively, an import like `use function Safe\foo;` can fail after a
  major bump if `foo` was dropped. Treat Safe major upgrades as a code change, not a
  patch, and re-run PHPStan.
- **Fixed autoload cost.** ~1000 functions across ~85 eagerly-loaded files add roughly
  700µs per request per the project's own measurement[^6]. Negligible for most apps;
  worth knowing for high-throughput, low-latency services without an opcache/preload
  warm path. Enabling OPcache preloading amortizes it.
- **Enforcement is only as good as your PHPStan coverage.** Any file, vendor path, or
  generated code not analyzed by PHPStan can still call the unsafe originals freely.
  On a large legacy codebase the rule is also initially noisy — expect a large first
  batch of violations.
- **Redundant on modern PHP for some functions.** On PHP 8.x several previously
  `false`-returning functions already throw. For those, Safe is belt-and-suspenders;
  the value concentrates in filesystem, JSON, string, and network functions that
  still return sentinels.

## When to Use / When Not

**Use when:**
- Your project already runs PHPStan in CI — the rule is what makes Safe enforceable.
- You want fail-fast semantics for filesystem, JSON, and I/O without hand-writing
  `=== false` checks on every call.
- You maintain a large PHP 8 codebase and want a mechanical path to consistent error
  handling.

**Avoid when:**
- You have no static analysis in the pipeline — Safe becomes opt-in trivia that's
  easy to forget.
- Your code already handles `false` returns explicitly and deliberately (Safe would
  fight that pattern; the Rector migration actively breaks it).
- You're on a minimal footprint where the eager per-request autoload of ~85 files is
  unwelcome and you cannot enable preloading.

## Alternatives

- azjezz/psl — a typed, exception-first standard library for PHP; broader rethink of
  the stdlib rather than a wrapper. Use instead when you want a coherent typed API,
  not a per-function opt-in over the existing globals.
- phpstan/phpstan — use alone when you want failure detection and stricter typing but
  do not want to route calls through a wrapper namespace.
- symfony/filesystem — use for filesystem operations specifically when you already
  live in the Symfony component ecosystem and want an object API that throws.
- Native PHP 8 — use nothing when your hot paths only touch functions PHP 8 already
  made throw; Safe adds little there.
- Manual `if ($x === false) throw ...` — use when only a handful of call sites matter
  and a library-wide dependency is not justified.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-09 | Repository created; wrappers generated from PHP docs, `Safe\` namespace[^1]. |
| 1.x | 2019 | PHP 7.2+ line; PHPStan rule and Rector migration config established. |
| 2.x | 2021 | PHP 8.0+ required; wrappers dropped for functions PHP 8 made throw natively. |

## References

[^1]: thecodingmachine/safe README and "Introducing Safe-PHP" release article. https://thecodingmachine.io/introducing-safe-php
[^2]: Generation from PHP documentation sources — see the `generator/` tooling and CONTRIBUTING. https://github.com/thecodingmachine/safe/blob/master/CONTRIBUTING.md
[^3]: PHPStan Safe rule. https://github.com/thecodingmachine/phpstan-safe-rule
[^4]: `Safe\Exceptions` classes and `SafeExceptionInterface`. https://github.com/thecodingmachine/safe/tree/master/lib/Exceptions
[^5]: Rector migration config `rector-migrate.php` and README "Automated refactoring" section (documents the manual error-handling caveat). https://github.com/thecodingmachine/safe
[^6]: README "Performance impact" section — ~700µs per request loading ~1000 functions from ~85 files. https://github.com/thecodingmachine/safe#performance-impact

## Tags

php, error-handling, exceptions, static-analysis, phpstan, code-generation, developer-tooling, library, composer
