# nikic/PHP-Parser

> A PHP parser written in PHP that turns source into a mutable AST — the tokenizer-plus-AST layer under most of the PHP static-analysis ecosystem.

[GitHub repo](https://github.com/nikic/PHP-Parser) ·
[License: BSD-3-Clause](https://github.com/nikic/PHP-Parser/blob/master/LICENSE)

## Overview

PHP-Parser parses PHP source into an abstract syntax tree, lets you traverse and
modify that tree, and prints it back to source. It has existed since 2011[^1] and
is maintained by Nikita Popov, a longtime PHP core contributor, which is a large
part of why its grammar tracks the language closely. It is a library, not a tool:
you build linters, refactoring engines, code generators, and metrics collectors
on top of it.

Its practical significance is downstream. PHPStan, Psalm, and Rector are all built
on PHP-Parser[^2], so a very large fraction of PHP static analysis in production
ultimately runs through this one AST. That gives the project outsized influence
and also sets the constraint it lives under: the AST shape is a public contract,
so node structure changes are breaking changes for an entire ecosystem, and the
maintainer is deliberately conservative about them.

The defining tension is that it is a PHP parser written in PHP, which must parse
language versions newer than the runtime it executes on. A tool running on PHP 7.4
still needs to parse PHP 8.4 syntax, and the host's native `token_get_all()` does
not know those newer tokens. PHP-Parser solves this with token emulation rather
than a native extension, which keeps it pure-PHP and installable anywhere Composer
runs, at the cost of carrying per-version lexer emulation logic.

## Getting Started

```bash
composer require nikic/php-parser
```

```php
<?php
use PhpParser\ParserFactory;
use PhpParser\NodeDumper;

$parser = (new ParserFactory())->createForNewestSupportedVersion();
$ast = $parser->parse('<?php echo "hi";');

echo (new NodeDumper)->dump($ast), "\n";
```

To rewrite code, add a visitor and pretty-print the result:

```php
use PhpParser\{Node, NodeTraverser, NodeVisitorAbstract, PrettyPrinter};
use PhpParser\Node\Stmt\Function_;

$traverser = new NodeTraverser();
$traverser->addVisitor(new class extends NodeVisitorAbstract {
    public function enterNode(Node $node) {
        if ($node instanceof Function_) {
            $node->stmts = []; // drop every function body
        }
    }
});

$ast = $traverser->traverse($ast);
echo (new PrettyPrinter\Standard)->prettyPrintFile($ast);
```

## Architecture / How It Works

The pipeline is lexer → parser → AST → (traverser) → pretty printer. The parser is
not hand-written: it is generated from an LALR(1) grammar (`grammar/php.y`) by a
patched build of `kmyacc`, then committed into the repo so consumers never run the
generator. This is why grammar changes come from the maintainer rather than casual
contributors — regenerating the tables is a specialized step.

Nodes live under `PhpParser\Node`, split into `Stmt` (statements), `Expr`
(expressions), `Scalar` (literals), and `Name`. Every node exposes typed public
properties (`$node->stmts`, `$node->expr`) plus an attribute bag carrying line
numbers, byte offsets, comments, and the raw tokens that produced it. Attributes,
not properties, are how positional and formatting metadata ride along.

Traversal is the visitor pattern: `enterNode`/`leaveNode` callbacks whose return
values steer the walk — return a replacement node to substitute, `REMOVE_NODE` to
delete, `DONT_TRAVERSE_CHILDREN` to prune, `STOP_TRAVERSAL` to bail. Several
visitors can run in one pass. Name resolution (`NodeVisitor\NameResolver`) and
parent/sibling linking are themselves just visitors you opt into.

Two subsystems are worth knowing. The **emulative lexer** synthesizes tokens for
syntax the host PHP is too old to tokenize (enums, readonly, `#[Attributes]`,
named args), so a tool on an older runtime can still parse newer code. The
**format-preserving pretty printer**, added in v4, diffs the modified AST against
the original token stream and reprints only changed nodes, leaving untouched code
byte-for-byte intact — the feature that makes surgical automated refactoring
(Rector) viable instead of reformatting whole files.

## Production Notes

- **Xdebug destroys parse throughput.** With Xdebug loaded, parsing large trees can
  be several times slower; the official performance guide's first recommendation is
  to disable it[^3]. This is the single most common "why is analysis slow" answer in
  the ecosystem.
- **Memory, not CPU, is the ceiling.** Each node is a full object with an attribute
  array. Parsing a large codebase allocates millions of objects; PHP's cyclic
  garbage collector churns on them, and peak memory — not wall-clock parse time — is
  usually what kills batch runs. Freeing ASTs between files and, where noted,
  disabling GC around hot loops are the documented mitigations.
- **The v4 → v5 upgrade has real API breaks.** `ParserFactory::create()` with the
  `PREFER_PHP7`/`ONLY_PHP7` constants was removed in favor of
  `createForNewestSupportedVersion()` / `createForHostVersion()` /
  `createForVersion()`[^4]. Lexer construction and how tokens are exposed also
  changed. Code that hardcoded the old factory call will not run on v5 unchanged.
- **The AST is a contract you don't control.** If you persist ASTs or match on
  specific node shapes, a minor language addition can add properties. Match
  defensively; don't assume a property set is closed.
- **Invalid input is a feature, not a crash.** With a collecting `ErrorHandler`, a
  syntactically broken file still yields a partial AST — essential for editor and
  language-server use cases, but it means "parsed" does not imply "valid."
- **Runtime vs. target-version are independent.** v5 runs on PHP ≥ 7.4 and parses
  PHP 7.0–8.4 (limited PHP 5.x); v4 runs on PHP ≥ 7.0 and parses PHP 5.2–8.3[^5].
  Pick the branch by both the PHP you run on and the PHP you need to read.

## When to Use / When Not

**Use when:**
- You need a real AST — refactoring, code generation, custom lint rules, metrics.
- You must parse PHP newer than the interpreter your tool runs on.
- You want formatting-preserving rewrites (change three lines, touch three lines).
- You're building on or extending PHPStan/Psalm/Rector and need the same tree they use.

**Avoid when:**
- Token-level analysis is enough — native `token_get_all()` / `PhpToken` is far
  lighter than building a full AST.
- You only emit PHP and never read it — a code generator like nette/php-generator
  is a better fit than an AST round-trip.
- You need error-tolerant incremental parsing for an editor hot path, where a parser
  designed for resilience may serve better.

## Alternatives

- microsoft/tolerant-php-parser — error-tolerant, position-preserving parser aimed at editors and language servers; use it when resilient, incremental parsing for IDE features matters more than a clean AST.
- glayzzle/php-parser — a PHP parser implemented in JavaScript; use it when you must parse PHP inside a Node.js toolchain.
- nette/php-generator — code generation only, no parsing; use it when you emit PHP but never need to read it back.
- php token_get_all / PhpToken — the native tokenizer; use it when token-level work suffices and you want to skip AST overhead entirely.
- rectorphp/rector — not an alternative but a consumer; reach for it when you want ready-made automated refactors instead of writing visitors yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-04 | First commit; grammar-generated parser[^1]. |
| 1.0 | 2014 | First stable release; traverser and pretty-printer APIs settle[^5]. |
| 2.0 | 2016 | PHP 7 parsing; dropped legacy runtimes[^5]. |
| 3.0 | 2017 | Node restructuring and API cleanup[^5]. |
| 4.0 | 2018 | Format-preserving pretty printer; PHP ≥ 7.0 runtime[^5]. |
| 5.0 | 2023 | ParserFactory redesign, lexer/token API changes; PHP ≥ 7.4 runtime, parses to PHP 8.x[^4][^5]. |

## References

[^1]: nikic/PHP-Parser repository, created 2011-04-18. https://github.com/nikic/PHP-Parser
[^2]: PHPStan, Psalm, and Rector declare `nikic/php-parser` as a dependency. https://github.com/rectorphp/rector / https://github.com/phpstan/phpstan-src
[^3]: PHP-Parser docs, "Performance" (disabling Xdebug, GC impact). https://github.com/nikic/PHP-Parser/blob/master/doc/component/Performance.markdown
[^4]: PHP-Parser v5 upgrading guide (ParserFactory and lexer changes). https://github.com/nikic/PHP-Parser/blob/master/UPGRADE-5.0.md
[^5]: PHP-Parser README and CHANGELOG (version/runtime support matrix). https://github.com/nikic/PHP-Parser/blob/master/CHANGELOG.md

## Tags

php, parser, ast, static-analysis, code-generation, refactoring, tokenizer, developer-tools, library, bsd-licensed
