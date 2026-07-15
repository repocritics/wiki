# peggyjs/peggy

> A PEG parser generator for JavaScript — the maintained successor to PEG.js, turning a grammar file into a standalone recursive-descent parser.

[GitHub repo](https://github.com/peggyjs/peggy) ·
[Official website](https://peggyjs.org/) ·
[License: MIT](https://github.com/peggyjs/peggy/blob/main/LICENSE)

## Overview

Peggy generates parsers from Parsing Expression Grammars (PEG)[^1]. You write a
grammar — a set of named rules combining literals, character classes, ordered
choice, repetition, and inline JavaScript semantic actions — and Peggy emits a
single self-contained `.js` file whose `parse(input)` either returns whatever
your actions build (an AST, a computed value, a transformed string) or throws a
syntax error with precise line/column information. It is scannerless: lexical
and syntactic analysis live in one grammar, so there is no separate tokenizer
step.

Peggy is a direct continuation of PEG.js, originally written by David Majda.
After PEG.js went unmaintained, the community forked it as Peggy in 2021;
version 1.0 was intentionally API-compatible with the last PEG.js release, so
migration is a rename of `require("pegjs")` to `require("peggy")`[^2]. It is now
maintained principally by Joe Hildebrand. With roughly 1.2k stars it is a
small-but-load-bearing project: not fashionable, but depended on across the JS
tooling ecosystem for DSLs, config-language parsers, and query parsers.

The defining tradeoff is PEG itself. Ordered choice makes grammars
unambiguous and gives unlimited lookahead via syntactic predicates, but it
forbids left recursion and silently commits to the first matching alternative.
That predictability is the reason to choose Peggy — and the source of its most
common surprises.

## Getting Started

```bash
npm install peggy          # library + `peggy` CLI
npx peggy arithmetic.peggy # emits arithmetic.js next to the grammar
```

```js
import * as peggy from "peggy";

const parser = peggy.generate(`
  // Left-associative add/subtract; actions compute the value directly.
  Expr = head:Term tail:(_ ("+" / "-") _ Term)* {
    return tail.reduce((acc, [, op, , n]) => op === "+" ? acc + n : acc - n, head);
  }
  Term = [0-9]+ { return parseInt(text(), 10); }
  _    = [ \\t]*
`);

parser.parse("1 + 2 - 3"); // => 0
```

On failure `parser.parse` throws a `SyntaxError` carrying `.location`
(start/end offset, line, column), `.expected`, and `.found`; call
`err.format([{ source, text }])` for a caret-annotated message.

## Architecture / How It Works

A grammar compiles through a fixed pass pipeline: parse the grammar into an AST,
run analysis/optimization passes, then generate JavaScript source via a small
bytecode-like intermediate representation. The output is a closed-over set of
`peg$parse<Rule>` functions implementing recursive descent — each PEG operator
maps to a concrete control-flow shape (sequence, ordered choice, `*`/`+`
loops, `&`/`!` predicates that match without consuming). Semantic actions are
inlined verbatim into the generated functions, which is why the emitted file is
readable but grows with grammar size.

`peggy.generate(grammar, opts)` is the whole surface. Useful options:
`output` (`"parser"`, `"source"`, or `"ast"`), `format` (`bare`, `commonjs`,
`es`, `amd`, `globals`, `umd`), `allowedStartRules` to expose more than one
entry rule, `plugins` to hook the pass pipeline, `trace` for a step tracer, and
`cache`.

`cache: true` is the packrat switch: it memoizes every rule result per input
position, converting PEG's worst case into guaranteed linear time at the cost
of substantial memory. It is off by default because typical grammars do not
backtrack pathologically and the memoization overhead makes them slower, not
faster. Turn it on only when profiling shows repeated re-parsing of the same
positions.

The generator is co-designed with the grammar language, so newer syntax (range
repetition like `expr|1..3|` and `expr|.., ","|` with a delimiter, grammar-level
`import`/`export` for splitting a grammar across files) is a matter of new
codegen shapes rather than a separate runtime. Generated parsers have no runtime
dependency on Peggy itself.

## Production Notes

- **No left recursion.** `Expr = Expr "+" Term / Term` recurses forever. You
  must rewrite left-recursive rules into repetition (`Term (op Term)*`) and
  rebuild associativity in the action. This is the single most common wall new
  users hit; it is inherent to PEG[^1], not a Peggy bug.
- **Ordered choice commits.** `"a" / "ab"` can never match `"ab"` — the first
  alternative succeeds on the `"a"` prefix and the rest is left unparsed. Order
  longer/more-specific alternatives first.
- **Error messages report the farthest failure, not the "intended" one.**
  Because choice is greedy and backtracking is silent, "Expected X but found Y"
  often names the last alternative tried at the deepest position, which can be
  confusing in large grammars. Naming rules with `Rule "human label" = ...`
  improves the reported `expected` set.
- **Whole-input, string-only.** There is no streaming or incremental parse; the
  entire input is held in memory as a string. Very large inputs (multi-hundred-MB
  logs) are a poor fit.
- **Bundle size.** Inlined actions plus per-rule functions make generated
  parsers larger than a hand-written tokenizer + parser for the same language;
  weigh this for browser bundles.
- **Precedence is manual.** There is no precedence-declaration feature; operator
  precedence is expressed by layering rules (factor → term → expression), which
  is verbose but explicit.
- **Types.** Peggy ships its own TypeScript definitions for the generation API;
  the generated parser is plain JavaScript, and recent CLI versions can emit a
  `.d.ts` for it, but rule return types are otherwise untyped `any`.

## When to Use / When Not

**Use when:**
- You are building a DSL, config language, query syntax, or template language
  and want an unambiguous grammar with readable errors.
- You want a zero-runtime-dependency parser you can commit as generated source.
- You are migrating an existing PEG.js grammar — Peggy is a drop-in successor.

**Avoid when:**
- Your grammar is naturally left-recursive or ambiguous (most published BNF for
  real programming languages) — an Earley or GLR parser will fit with less
  rewriting.
- You need streaming/incremental parsing or to parse binary formats.
- You need multi-target output (Java, Go, Python parsers from one grammar).

## Alternatives

- kach/nearley — Earley parser for JS; accepts ambiguous and left-recursive
  grammars. Use when PEG's ordered choice fights your grammar.
- Chevrotain/chevrotain — no code generation; a fast parsing DSL you write
  directly in TypeScript. Use when you want debuggable, hand-tunable parsers.
- ohmjs/ohm — also PEG-based, but separates grammar from semantics and supports
  incremental parsing. Use when you want cleaner grammar/action separation.
- antlr/antlr4 — LL(*) generator with many target languages. Use when you need
  parsers in more than just JavaScript, or tooling maturity for big grammars.
- pegjs/pegjs — the unmaintained predecessor. Use only to read legacy grammars;
  new work should be on Peggy.

## History

| Version | Date | Notes |
|---------|------|-------|
| PEG.js (prior) | 2010–2016 | David Majda's original generator; basis for the fork[^2]. |
| 1.0.0 | 2021-04-16 | Peggy fork; API-compatible with last PEG.js, bundled TS types[^2]. |
| 2.0.0 | 2022-05-28 | Grammar-language and plugin-API evolution. |
| 3.0.0 | 2023-02-22 | Source map support and generator improvements. |
| 4.0.0 | 2024-02-13 | Grammar-level `import`/`export` for multi-file grammars. |
| 5.0.0 | 2025-05-03 | Dropped end-of-life Node versions; toolchain modernization. |
| 5.1.0 | 2026-03-01 | Latest release as of this writing. |

## References

[^1]: Bryan Ford, "Parsing Expression Grammars: A Recognition-Based Syntactic Foundation" (POPL 2004) — the PEG formalism. https://bford.info/pub/lang/peg/
[^2]: Peggy README and PEG.js migration guide. https://github.com/peggyjs/peggy
[^3]: Peggy documentation. https://peggyjs.org/documentation.html
[^4]: Peggy releases (version dates). https://github.com/peggyjs/peggy/releases

## Tags

javascript, parser-generator, peg, parsing-expression-grammar, compiler, dsl, recursive-descent, pegjs, grammar, ast, code-generation
