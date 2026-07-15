# Chevrotain/chevrotain

> A parser-building toolkit for JavaScript/TypeScript where the grammar is ordinary code — no grammar file, no code-generation step.

[GitHub repo](https://github.com/Chevrotain/chevrotain) ·
[Official website](https://chevrotain.io) ·
[License: Apache-2.0](https://github.com/Chevrotain/chevrotain/blob/master/LICENSE)

## Overview

Chevrotain is a parsing library, not a parser generator. Where ANTLR, Peggy, or
Jison read a grammar file and emit source code, Chevrotain has you write the
grammar directly as method calls inside a TypeScript/JavaScript class[^1]. There
is no `.g4` file, no build step that regenerates a parser, and no generated code
to check into your repo. The grammar *is* the program.

That single decision drives everything else about the library. Because rules are
plain functions calling `this.CONSUME(...)`, `this.SUBRULE(...)`, `this.OR(...)`,
you get IDE navigation, breakpoints, and refactoring on your grammar for free,
and you dispatch through direct function calls rather than a generated state
table — which is the main reason Chevrotain is fast relative to generated JS
parsers[^2]. The cost is that a grammar expressed as imperative code is less
declarative than a `.g4` file: left-recursion must be rewritten by hand, and the
grammar's shape is only knowable by *running* it (see Architecture).

Chevrotain targets LL(k) grammars with configurable lookahead, plus gate
predicates and backtracking for the hard cases; true ALL(\*) lookahead is
available through the separate `chevrotain-allstar` plugin maintained by the
Langium team. It is used in production by Langium (the LSP language-engineering
framework), HyperFormula (Handsontable's spreadsheet formula engine),
prettier-java, and JHipster's JDL — parsers where a code-generation step in the
build would be an operational liability. The library was created by Shahar Soel.

## Getting Started

```bash
npm install chevrotain
```

A complete lexer + parser that recognizes `1 + 2 + 3` and returns a CST:

```ts
import { createToken, Lexer, CstParser } from "chevrotain";

const Int = createToken({ name: "Int", pattern: /\d+/ });
const Plus = createToken({ name: "Plus", pattern: /\+/ });
const WhiteSpace = createToken({
  name: "WhiteSpace",
  pattern: /\s+/,
  group: Lexer.SKIPPED,
});

const tokens = [WhiteSpace, Int, Plus];
const lexer = new Lexer(tokens);

class Calc extends CstParser {
  constructor() {
    super(tokens);
    this.performSelfAnalysis(); // grammar is analyzed once, here
  }

  expression = this.RULE("expression", () => {
    this.CONSUME(Int);
    this.MANY(() => {
      this.CONSUME(Plus);
      this.CONSUME2(Int); // note the suffix: two CONSUME(Int) in one rule
    });
  });
}

const parser = new Calc();

export function parse(text: string) {
  const { tokens: lexed } = lexer.tokenize(text);
  parser.input = lexed;
  const cst = parser.expression();
  if (parser.errors.length > 0) throw new Error(parser.errors[0].message);
  return cst;
}
```

Note the `CONSUME2` suffix: because rules are functions, Chevrotain needs a
stable per-call-site identity, so consuming the same token type twice in one
rule must number the calls (`CONSUME`, `CONSUME2`, …) — likewise for `SUBRULE`,
`OR`, `MANY`, `OPTION`. This is the most common early stumbling block.

## Architecture / How It Works

The pipeline is two decoupled stages: a **Lexer** turns source text into a flat
token stream, and a **Parser** consumes that stream. They share nothing but the
token vocabulary, so you can swap either independently.

The defining internal mechanism is **grammar recording**. When you call
`performSelfAnalysis()` in the constructor, Chevrotain runs each rule *once* in a
special recording mode where `CONSUME`/`SUBRULE`/`OR` are no-ops that record
structure instead of parsing input. From that traversal it reconstructs the full
grammar as data — which is what powers computed lookahead, ambiguity detection,
error messages, syntax-diagram generation, and content-assist. This is why the
grammar must be *runnable* code and why self-analysis happens exactly once per
parser class, not per parse.

Two authoring styles exist. `CstParser` builds a Concrete Syntax Tree
automatically; you then walk it with a generated visitor
(`getBaseCstVisitorConstructor()`). `EmbeddedActionsParser` lets you interleave
semantic actions and build any value directly during the parse. CST + visitor is
the recommended default because it keeps grammar and semantics separate and
survives grammar edits better; embedded actions are faster and terser but couple
tree-shape to logic.

**Error recovery** is built in and on by default: Chevrotain implements
single-token insertion/deletion and re-synchronization so a parser can report
multiple errors in one pass rather than dying on the first. Errors accumulate on
`parser.errors` rather than throwing.

## Production Notes

- **Instantiate the parser once, reuse it.** `performSelfAnalysis()` is real
  work; constructing a new parser per request is a common and avoidable
  performance mistake. Create one instance, then reset it by assigning
  `parser.input` before each parse.
- **The `CONSUME2`/`OR2` numbering is load-bearing, not cosmetic.** Duplicate
  numbers at different call sites silently corrupt lookahead. Chevrotain's
  self-analysis catches many such mistakes at construction time and throws — read
  those errors literally; they name the offending rule.
- **No left recursion.** Being LL, direct or indirect left-recursive rules must
  be rewritten iteratively (typically with `MANY`), as in the example above.
  This trips up people porting a yacc/ANTLR grammar.
- **Lexer token order matters.** `createToken` patterns are tried in array order;
  put longer/more-specific tokens before their prefixes, or use `longer_alt` and
  token categories. Keyword-vs-identifier collisions are the classic bug.
- **Lookahead cost.** Very high `maxLookahead` or heavy backtracking/gate
  predicates can dominate parse time. Prefer restructuring the grammar or the
  `chevrotain-allstar` plugin over cranking lookahead globally.
- **Bundle size.** Chevrotain is a runtime dependency shipped with your app, not
  a build-time generator — the library ships to the browser. For size-sensitive
  frontends that is a real tradeoff versus generated parsers with no runtime lib.
- **Legacy runtimes are unsupported.** Modern versions target current Node LTS
  and evergreen browsers and ship ESM; older targets need transpilation and
  polyfills. Check the package `engines` field before committing.

## When to Use / When Not

**Use when:**
- You want the grammar to live in your codebase as debuggable, testable code
  with no generation step in the build.
- You need parse speed close to a hand-written recursive-descent parser.
- You need robust, multi-error reporting and error recovery out of the box.
- You're building tooling (formatters, linters, LSP servers) where the grammar
  evolves continuously.

**Avoid when:**
- You want a compact, declarative grammar file to hand to non-JS tools, or the
  same grammar reused across languages — a generator like ANTLR fits better.
- Your grammar is naturally left-recursive or ambiguous and you'd rather not
  restructure it (consider an Earley or GLR parser).
- You need zero runtime dependency / minimal bundle and are fine with generated
  code instead.

## Alternatives

- antlr/antlr4 — mature LL(\*) generator with a declarative grammar and many
  target languages; use when you need one grammar shared across JVM/JS/Python or
  prefer a `.g4` file over a JS API.
- peggyjs/peggy — PEG grammar file with code generation (the maintained PEG.js
  successor); use when you want a declarative grammar and don't mind a build step.
- kach/nearley — Earley parser that handles ambiguous and left-recursive
  grammars directly; use when your grammar is genuinely ambiguous.
- ohmjs/ohm — PEG-based, keeps grammar strictly separate from semantic actions;
  use when you value that separation over raw speed.
- no-context/moo — a fast tokenizer only; use when you need lexing but are
  hand-writing the parser yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-04 | Repository created; library origins at SAP[^1]. |
| 4.x | 2018 | CST output mode and the visitor pattern established as the recommended path. |
| 7.x | 2020 | API cleanup, removal of long-deprecated methods. |
| 9.x | 2021 | Dropped legacy-browser support; modernized build. |
| 10.x | 2022 | Internal TypeScript modernization, ESM distribution. |
| 11.x–12.x | 2023–2026 | Ongoing maintenance; current major line. Last pushed 2026-07[^3]. |

## References

[^1]: Chevrotain README and FAQ — "grammars are written as pure JavaScript sources without a code generation phase." https://chevrotain.io/docs/FAQ.html
[^2]: Chevrotain performance documentation and benchmark. https://chevrotain.io/performance/
[^3]: GitHub API metadata for Chevrotain/chevrotain (stars, license, last push) — retrieved 2026-07. https://github.com/Chevrotain/chevrotain

## Tags

parser, lexer, tokenizer, parsing, typescript, javascript, parser-library, grammars, compiler, recursive-descent, ll-parser
