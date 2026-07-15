# pest-parser/pest

> A general-purpose parser generator for Rust that reads Parsing Expression Grammars from separate `.pest` files.

[GitHub repo](https://github.com/pest-parser/pest) ·
[Official website](https://pest.rs) ·
[License: Apache-2.0](https://github.com/pest-parser/pest/blob/master/LICENSE-APACHE)

## Overview

pest is a parser generator for Rust built around Parsing Expression Grammars (PEG). You write a grammar in a dedicated `.pest` file, attach it to a struct with a derive macro, and pest generates a parser that turns input into an iterator of typed token pairs[^1]. The design goal, stated in the README, is "accessibility, correctness, and performance" — in practice the defining choice is that grammar and procedural code stay in separate files, so the grammar reads as a standalone formalization of the language rather than being interleaved with Rust.

PEG is the model that shapes everything else. Unlike context-free grammars (used by yacc/bison-style tools), PEG rules are ordered-choice and greedy: the `|` operator commits to the first matching alternative and never backtracks across it, so a PEG cannot be ambiguous but can silently match less than you expected if alternatives are ordered wrong. This makes grammars deterministic and easy to reason about locally, at the cost of the "prioritized choice" footgun that trips up newcomers coming from regex or CFG tools.

pest sits in the same niche as `nom` and `lalrpop` but occupies a distinct point in it: `nom` is a parser-combinator library (parsers are Rust functions, no grammar file), `lalrpop` is an LR(1) generator with a CFG grammar, and pest is a PEG generator with an external grammar. Its ergonomics — readable grammar files, automatic error messages with source spans, an online grammar editor[^2] — are the reason it is widely used in real projects (Prisma, comrak, tera, ZoKrates, and Vector all depend on it). As of 2026 the repository has roughly 5,400 stars and is actively maintained, with the most recent commits within days of this writing.

## Getting Started

Add both crates — `pest` is the runtime, `pest_derive` provides the `#[derive(Parser)]` macro:

```bash
cargo add pest pest_derive
```

Write a grammar file `src/ident.pest`:

```pest
alpha = { 'a'..'z' | 'A'..'Z' }
digit = { '0'..'9' }
ident = { !digit ~ (alpha | digit)+ }
ident_list = _{ ident ~ (" " ~ ident)* }   // leading _ = silent rule, emits no token
```

Derive a parser and iterate the pairs:

```rust
use pest::Parser;
use pest_derive::Parser;

#[derive(Parser)]
#[grammar = "ident.pest"]
struct IdentParser;

fn main() {
    let pairs = IdentParser::parse(Rule::ident_list, "a1 b2")
        .unwrap_or_else(|e| panic!("{e}"));

    for pair in pairs {
        println!("{:?}: {}", pair.as_rule(), pair.as_str());
        for inner in pair.into_inner() {
            println!("  {:?}: {}", inner.as_rule(), inner.as_str());
        }
    }
}
```

Grammar syntax operators: `~` sequence, `|` ordered choice, `*`/`+`/`?` repetition, `!`/`&` negative/positive lookahead, `_` prefix for silent rules, `@` for atomic (no implicit whitespace), `$` for compound-atomic.

## Architecture / How It Works

pest is a workspace of several crates with a bootstrapping story at its center:

- **`pest`** — the runtime: the `Parser` trait, the `Pairs`/`Pair`/`Span` token API, error types, and the precedence-climbing helper (`pest::pratt_parser`).
- **`pest_meta`** — parses and validates `.pest` grammar files. Its own grammar (`meta/src/grammar.pest`) is written in pest and bootstrapped: an older pest parses the grammar that defines pest[^3].
- **`pest_generator`** — turns a validated grammar into Rust source (the `Rule` enum plus the state machine).
- **`pest_derive`** — the proc-macro front end that reads the `#[grammar = "..."]` path at compile time and invokes the generator.

The generated parser is a recursive PEG matcher driven by the `ParserState` type. Each rule becomes a function that advances a cursor and, on success, pushes a token pair; on failure it rewinds. Because PEG is greedy with ordered choice, there is no separate lexer — tokenization and parsing are the same pass, and whitespace/comment handling is folded in via the special `WHITESPACE` and `COMMENT` rules that pest implicitly inserts between tokens (except inside atomic rules).

The coupling that matters most in practice: **the grammar is resolved at compile time by the derive macro**, so a grammar edit forces a recompile, and grammar errors surface as macro-expansion errors rather than runtime errors. This is a feature (invalid grammars can't ship) but means grammar iteration speed is bounded by Rust compile times, and the `.pest` file must be findable relative to the crate root (or supplied inline via `#[grammar_inline = "..."]`).

Error reporting is a first-class part of the design. On a parse failure pest produces a `pest::error::Error` that, via its `Display` impl, renders a caret-annotated source snippet with the positives/negatives it expected. This comes for free from the ordered-choice model — the parser knows exactly which rules it tried at the failure position.

## Production Notes

**Backtracking cost.** PEG's ordered choice means a rule can attempt many alternatives at the same position, and pest does not memoize by default (it is not a packrat parser). Pathological grammars with heavy alternation or deep optional nesting can degrade toward exponential time on adversarial input. Structure hot rules so the common alternative comes first, and prefer lookahead pruning (`!x ~ ...`) over blind alternation.

**Left recursion is not allowed.** Like all PEG tools, pest cannot express a directly or indirectly left-recursive rule — it will loop. Arithmetic and other left-associative grammars must be rewritten with repetition plus the `pratt_parser` precedence climber, which is the idiomatic pest way to handle operator precedence.

**Whitespace surprises.** The implicit `WHITESPACE` rule is inserted between every `~` in a non-atomic rule. Forgetting to mark a rule atomic (`@`) when you need exact matching (identifiers, string literals, numbers) is the single most common bug — tokens silently absorb surrounding whitespace. Conversely, defining `WHITESPACE` and then wondering why a rule won't match usually means the rule should have been atomic.

**Compile-time weight.** `pest_derive` runs the full generator inside the proc-macro during every build of the dependent crate; large grammars add measurable compile time and the generated `Rule` enum plus match arms can be sizable. There is no ahead-of-time codegen-to-file workflow in the standard flow.

**MSRV and no_std.** The library targets a Minimum Supported Rust Version of **1.83.0**[^4]. Both `pest` and `pest_derive` support `no_std` when built with `default-features = false`, which pulls in `alloc` and drops `std`-only conveniences — usable on embedded targets such as `thumbv7em-none-eabihf`.

**`Rule` enum name collisions.** The derive always generates an enum named `Rule`. Defining two `#[derive(Parser)]` structs in the same module collides; the documented workaround is to place each parser struct in its own module.

**2.x is the current line.** The crate is on the 2.x series on crates.io; pin to `"2"` and expect semver-compatible updates. The 1.x → 2.x transition years ago changed the grammar syntax and API significantly, so old tutorials referencing 1.x syntax do not apply.

## When to Use / When Not

**Use when:**
- You want a readable, standalone grammar file separated from parsing logic.
- You're parsing a config format, DSL, or query language where good error messages matter.
- You want automatic source-span error reporting without hand-writing it.
- Your grammar is naturally expressible as PEG (most hand-designed languages are).

**Avoid when:**
- You need the last measure of throughput on huge inputs — a hand-written or combinator parser (nom) usually wins on tight loops and gives finer control over allocation.
- Your grammar is heavily left-recursive or ambiguity-tolerant — an LR/GLR tool (lalrpop, tree-sitter) is a more natural fit.
- You want to build the AST directly during parsing — pest hands you an untyped pair tree that you must walk and convert yourself (see the `pest-typed`/`pest_consume` ecosystem for typed alternatives).
- You need incremental/error-recovering parsing for an editor — tree-sitter is designed for that; pest is not.

## Alternatives

- rust-bakery/nom — parser-combinator library; parsers are Rust functions, no grammar file. Use it when you want maximum control over performance and allocation and don't mind writing the parser in code.
- lalrpop/lalrpop — LR(1) parser generator with a CFG grammar and typed actions. Use it when your grammar is left-recursive or you want the parser to build a typed AST directly.
- tree-sitter/tree-sitter — incremental, error-recovering GLR parser generator with bindings for many languages. Use it for editor tooling, syntax highlighting, or non-Rust consumers.
- kevinmehall/rust-peg — another PEG generator using an inline macro grammar. Use it when you prefer the grammar embedded in Rust source over a separate file.
- zesterer/chumsky — combinator parser with strong error recovery and a functional API. Use it when you want nom-style composition plus good diagnostics.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2016-04 | Repository created; PEG parser generator for Rust[^5]. |
| 1.0 | 2017 | First stable line; original grammar syntax and API. |
| 2.0 | 2018 | Major rewrite: new grammar syntax, redesigned pairs API, workspace split. |
| 2.x | 2019–2026 | Ongoing 2.x series: `pratt_parser`, `no_std` support, MSRV moved to 1.83.0[^4]. |

## References

[^1]: pest README, "pest. The Elegant Parser." https://github.com/pest-parser/pest
[^2]: pest online grammar editor / fiddle. https://pest.rs/#editor
[^3]: Bootstrapped meta-grammar, `meta/src/grammar.pest`. https://github.com/pest-parser/pest/blob/master/meta/src/grammar.pest
[^4]: pest README, "Minimum Supported Rust Version (MSRV)" — Rust 1.83.0. https://github.com/pest-parser/pest#minimum-supported-rust-version-msrv
[^5]: GitHub repository metadata, created 2016-04-24. https://github.com/pest-parser/pest

## Tags

rust, parser, parser-generator, peg, parsing-expression-grammar, grammar, dsl, compiler, proc-macro, no-std
