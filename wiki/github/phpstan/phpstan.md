# phpstan/phpstan

> Static analysis for PHP that finds type errors and dead code without running the program — driven by opt-in strictness "levels".

[GitHub repo](https://github.com/phpstan/phpstan) ·
[Official website](https://phpstan.org/) ·
[License: MIT](https://github.com/phpstan/phpstan/blob/master/LICENSE)

## Overview

PHPStan is a static analyzer for PHP: it parses your code into an AST, infers
types, and reports errors (undefined methods, wrong argument types, unreachable
branches, always-false conditions) without executing anything. Started by Ondřej
Mirtes in 2016, it reached a stable 1.0 in November 2021 and 2.0 in November
2024[^1][^2]. It is one of the two dominant PHP analyzers alongside Psalm, and
in a language with gradual, largely optional typing it functions as the missing
compiler front-end.

Its defining idea is the **rule level**: analysis strictness is a dial from 0
(loosest) to 10 (strictest), so a legacy codebase can adopt PHPStan at level 0,
pass, and ratchet upward one level at a time rather than facing thousands of
errors on day one[^3]. This gradualism — plus the baseline mechanism (below) —
is why PHPStan spread through existing, untyped PHP codebases rather than only
greenfield ones.

The GitHub repo you are looking at (`phpstan/phpstan`) is the **distribution**
repo: it ships a prebuilt, dependency-scoped PHAR and the Composer package most
people install. The actual source lives in `phpstan/phpstan-src` and is where
pull requests go[^4]. That split matters when you try to read or contribute to
the code.

## Getting Started

```bash
composer require --dev phpstan/phpstan
vendor/bin/phpstan analyse src tests
```

A `phpstan.neon` (or `phpstan.neon.dist`) config in NEON format drives most
projects:

```neon
parameters:
    level: 6
    paths:
        - src
        - tests
    excludePaths:
        - src/Legacy/*
```

Bump `level` upward as the codebase gets cleaner; `level: max` is an alias for
the highest currently defined level, so pinning a number is safer for
reproducible CI. Try rules interactively at the online playground
(phpstan.org/try) without installing anything.

## Architecture / How It Works

PHPStan builds on `nikic/php-parser` for the AST[^5] and adds its own type
inference engine on top. The pipeline, roughly:

- **Scope** — as the analyzer walks statements it maintains a `Scope` object
  holding the inferred type of every variable at that program point, narrowed by
  conditions (`if ($x instanceof Foo)` narrows `$x` inside the branch). This
  flow-sensitive typing is the core of what PHPStan does.
- **Rules** — each check is a `Rule` bound to an AST node type. Levels are just
  bundles of rules; raising the level enables more rules and stricter type
  compatibility checks (nullability at ~level 6-7, `mixed` handling at 9-10).
- **Reflection & PHPDoc** — types come from native reflection plus PHPDoc
  annotations (`@param`, `@return`, `@var`, generics via `@template`). PHPStan's
  PHPDoc type language is far richer than PHP's native types (array shapes,
  integer ranges, literal strings, conditional return types).
- **Extensions** — because PHPStan never runs your code, dynamic behavior is
  invisible to it. Dynamic-return-type extensions, method/property reflection
  extensions, and stub files teach it about magic methods, container `get()`
  calls, ORM entities, and framework DSLs. Ecosystem packages
  (`phpstan-phpunit`, `phpstan-doctrine`, `phpstan-symfony`, Larastan for
  Laravel) are largely bundles of these extensions.
- **Result cache** — analysis results are cached per file and dependency graph,
  so re-runs only re-analyze changed files and their dependents. This is what
  makes it usable in a watch/CI loop on large codebases.

The distributed PHAR uses `humbug/php-scoper` to prefix all vendored
dependencies, isolating PHPStan's copy of `php-parser` from your project's. The
upside is zero dependency conflicts; the downside is that extensions must be
loaded in a way compatible with that scoping, and you cannot simply reach into
PHPStan internals from your own autoloader.

## Production Notes

**Baseline is the real adoption tool.** `phpstan analyse --generate-baseline`
writes a `phpstan-baseline.neon` that suppresses every currently-reported error,
so you can enforce "no new errors" in CI immediately and burn down the baseline
over time[^6]. Teams that skip this and try to fix everything up front usually
stall.

**It analyzes the PHP version you tell it to, not the one you run.** Set
`phpVersion` in config; otherwise inference uses the CLI PHP version, and
behavior differences (e.g. functions that return `false` vs throw between
versions) can produce surprising results.

**False positives are a fact of life on dynamic code.** Magic `__get`,
service containers, `array` soup, and framework conventions all produce errors
that are technically correct ("PHPStan can't see this") but not real bugs. The
fix hierarchy is: install the relevant extension → add stubs → `ignoreErrors`
regex in config → baseline as last resort. Reaching for `@phpstan-ignore` inline
comments everywhere is a smell.

**Level bumps are where the work is.** Each level increment on a large codebase
can surface hundreds to thousands of new errors. Treat level increases as
scheduled projects, not routine upgrades. The `bleedingEdge` include opts into
rules that will become default in the next major version — useful for staying
ahead, but it will make CI stricter without a version bump.

**Memory.** Large monorepos routinely need `--memory-limit=-1` or a raised PHP
`memory_limit`; the result cache mitigates repeat cost but the cold analysis of
a big codebase is memory-hungry.

**Reading the source is a two-repo affair.** Bug reports and PRs belong in
`phpstan/phpstan-src`; the issue tracker on `phpstan/phpstan` is the umbrella
where users file everything, which is why its open-issue count runs high.

## When to Use / When Not

**Use when:**
- You have PHP (typed or legacy) and want a type checker in CI that can start
  loose and tighten gradually.
- You want the widest framework extension ecosystem (Symfony, Doctrine, Laravel
  via Larastan, PHPUnit).
- You need rich PHPDoc-based generics and array-shape types that native PHP
  syntax cannot express.

**Avoid / reconsider when:**
- You want taint/security analysis as a first-class feature — Psalm's taint
  analysis is more mature there.
- Your codebase is so dynamic (heavy magic, no PHPDoc) that you'd spend more
  time writing stubs than fixing bugs — budget for it honestly.
- You want automated fixing: PHPStan reports, it does not rewrite. Pair it with
  Rector for transformations.

## Alternatives

- vimeo/psalm — the other major PHP analyzer; use instead when you specifically
  want taint analysis or prefer its `@psalm-` annotation dialect.
- phan/phan — analyzer requiring the `php-ast` C extension; use when you want
  ast-extension-backed speed and don't need PHPStan's extension ecosystem.
- rectorphp/rector — same-ecosystem author's tool; use *alongside* PHPStan when
  you want automated refactoring/upgrades, not just detection.
- phpmd/phpmd — mess/complexity detection; use for cyclomatic-complexity and
  code-smell rules rather than type correctness.
- friendsofphp/php-cs-fixer — style/formatting only; complementary, not a
  substitute for type analysis.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016-12 | First public release[^1]. |
| 0.12 | 2019-12 | Long-lived pre-1.0 series; widespread adoption. |
| 1.0 | 2021-11-01 | First stable; backward-compat guarantees[^1]. |
| 1.11 | 2024 | Level 10 introduced; stricter `mixed` handling. |
| 2.0 | 2024-11 | Second major; dropped older PHP support, tightened defaults[^2]. |
| 2.2.x | 2026 (current) | Active default branch. |

## References

[^1]: PHPStan blog, "PHPStan 1.0 Released!" — 2021-11-01. https://phpstan.org/blog/phpstan-1-0-released
[^2]: PHPStan blog, "PHPStan 2.0 Released!" — 2024-11. https://phpstan.org/blog/phpstan-2-0-released
[^3]: PHPStan docs, "Rule Levels". https://phpstan.org/user-guide/rule-levels
[^4]: `phpstan/phpstan-src` — the source repository where PRs are accepted. https://github.com/phpstan/phpstan-src
[^5]: `nikic/php-parser` — the PHP AST library PHPStan is built on. https://github.com/nikic/PHP-Parser
[^6]: PHPStan docs, "The Baseline". https://phpstan.org/user-guide/baseline

## Tags

php, static-analysis, type-checking, linter, developer-tools, ci, code-quality, phpstan, cli
