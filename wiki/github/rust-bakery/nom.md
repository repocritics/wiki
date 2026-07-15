# rust-bakery/nom

> A parser-combinator library for Rust that builds parsers from small typed functions instead of a grammar file, with zero-copy slices and optional streaming support.

[GitHub repo](https://github.com/rust-bakery/nom) ·
[Docs](https://docs.rs/nom) ·
[License: MIT](https://github.com/rust-bakery/nom/blob/main/LICENSE)

## Overview

nom is a parser-combinator library, originally written by Geoffroy Couprie ("Geal") and now maintained under the `rust-bakery` organization[^1]. Rather than declaring a grammar in a separate file and generating code (the lex/yacc model), you compose small parser functions — "take 5 bytes", "match the tag `HTTP`", "one or more digits" — into larger ones. The composed code ends up looking close to the grammar it implements, and every piece is an ordinary Rust value that can be unit-tested in isolation.

Its original and still strongest domain is binary formats: network protocols, media containers, on-disk structures. The core input type is `&[u8]`, parsers return sub-slices of the input without copying, and there is genuine support for streaming (partial) input, where a parser can report that it needs more data rather than returning a wrong answer. It works on `&str` and text formats too, and has been used to prototype programming-language parsers, but binary and network parsing is where its design choices pay off. With roughly 10.4k stars and a very wide reverse-dependency graph (the `rusticata` protocol parsers, `x509-parser`, `der-parser`, and many others build on it), it is the most established parser library in the Rust ecosystem[^2].

The defining tension is API churn. nom has rewritten its core interface twice in ways that invalidated most existing tutorials: the macro-to-function migration around v5 (2019) and the `Parser`-trait redesign in v8 (2025). The library itself is stable and heavily fuzzed; the risk is that documentation you find online frequently describes an API that no longer exists.

## Getting Started

```toml
[dependencies]
nom = "8"
```

A hexadecimal color parser (nom 8 API, using the `Parser` trait's `.parse()` method):

```rust
use nom::{
    bytes::complete::{tag, take_while_m_n},
    combinator::map_res,
    IResult, Parser,
};

#[derive(Debug, PartialEq)]
pub struct Color { pub red: u8, pub green: u8, pub blue: u8 }

fn hex_primary(input: &str) -> IResult<&str, u8> {
    map_res(
        take_while_m_n(2, 2, |c: char| c.is_ascii_hexdigit()),
        |s| u8::from_str_radix(s, 16),
    ).parse(input)
}

fn hex_color(input: &str) -> IResult<&str, Color> {
    let (input, _) = tag("#")(input)?;
    let (input, (red, green, blue)) =
        (hex_primary, hex_primary, hex_primary).parse(input)?;
    Ok((input, Color { red, green, blue }))
}
```

`no_std` builds are supported via feature flags: disabling default features drops `std`, and the `alloc` feature gates combinators that allocate (`many0`, `many1`, etc.).

## Architecture / How It Works

The whole library is built on one return type:

```rust
type IResult<I, O, E = (I, ErrorKind)> = Result<(I, O), Err<E>>;
```

A parser is anything callable that takes input `I` and returns either `Ok((remaining_input, output))` or an error. Because success carries the *remaining* input, parsers chain by threading that leftover forward — this is the entire combinator mechanism.

`Err<E>` has three variants, and understanding them is the key to using nom correctly:

- `Err::Error(e)` — a recoverable failure. Combinators like `alt` catch this and try the next branch.
- `Err::Failure(e)` — an unrecoverable failure. It short-circuits past `alt`; used with `cut` to commit to a branch and produce a real error message instead of a confusing "no branch matched".
- `Err::Incomplete(needed)` — streaming only: the parser ran out of input before deciding.

Every basic combinator exists in two modules, `nom::bytes::complete` / `nom::bytes::streaming` (and likewise for `character`, `number`, etc.). The **complete** variants treat end-of-input as a normal error; the **streaming** variants return `Incomplete` so a caller feeding data in chunks can wait for more. Choosing the wrong module is the single most common beginner mistake — streaming parsers on a fully-buffered slice yield spurious `Incomplete` errors.

Since nom 8, combinators are unified under the `Parser` trait, and tuples of parsers implement `Parser` directly (the older free-standing `tuple()` combinator is deprecated). Parsers are invoked with `.parse(input)` and combinators can be chained as methods. Error types are generic over the `ParseError` trait: the default `(I, ErrorKind)` is terse and cheap; `VerboseError` accumulates a context stack at some cost; you can supply your own type. There is no separate lexer/tokenizer stage — whitespace handling and AST construction happen inline in the same parsers.

## Production Notes

**Documentation rot from API rewrites.** The biggest practical hazard is not runtime behavior, it is search results. Macros that defined nom for years — `named!`, `do_parse!`, `ws!`, `alt!` — were removed in the v5 function migration. StackOverflow answers, blog posts, and even some crates' examples still use them. Then nom 8 moved to the `Parser` trait, so v5–v7 function examples that call `tuple(...)` or expect combinators as plain `Fn` also drift. When copying example code, check which major version it targets first.

**Streaming vs complete.** Covered above; worth repeating because it is the top source of "why does my parser return Incomplete" issues. Most applications that hold the whole input in memory want the `complete` submodules everywhere.

**Error ergonomics.** Default errors point at the failing input slice and an `ErrorKind`, which is enough to branch on but not human-readable. Producing good messages requires `VerboseError` plus `context`/`cut`, or the external `nom_locate` crate to recover line/column. Budget real effort here; nom gives you the mechanism but not friendly errors out of the box.

**Compile times and type bloat.** Deeply nested combinators build large, monomorphized types. This inflates compile times and, more painfully, produces enormous type-mismatch error messages when a combinator's input/output types don't line up. Factoring sub-parsers into named `fn`s with explicit signatures (rather than one giant nested expression) keeps both the compiler and the errors manageable.

**MSRV.** The 8.0 series requires rustc 1.65 or newer, and the policy is to only raise the minimum on a major-version bump[^3].

**Safety.** Parsers are `#![forbid(unsafe_code)]`-friendly in spirit and routinely fuzzed; the maintainers note that fuzzing has historically found bugs only in code written outside nom, not in nom's combinators. For untrusted binary input this is a meaningful advantage over hand-written C parsers.

## When to Use / When Not

**Use when:**
- You are parsing a binary or network format and want zero-copy slices plus real streaming support.
- You need incremental parsing over chunked input (protocols, huge files) with correct "need more data" behavior.
- You want parsers that are ordinary testable functions, no build-time codegen step.
- You are building on the existing nom-based protocol ecosystem (`rusticata`, `der-parser`, etc.).

**Avoid when:**
- You want readable, maintainable *error messages* for a programming language with minimal effort — `chumsky` or `pest` give better ergonomics.
- You prefer a declarative grammar in a separate file — `pest` (PEG) or `lalrpop` (LR) fit that model.
- You want a stable long-lived API — nom's periodic core rewrites impose migration work.
- Your grammar is a conventional programming language where a generated LR/LL parser would be clearer.

## Alternatives

- rust-bakery/winnow — a fork of nom (by Ed Page) with a reworked API and faster iteration; used by `toml_edit`. Choose it when you want nom's model but a more actively evolving interface.
- zesterer/chumsky — combinator library focused on error recovery and diagnostics; choose it when parsing programming languages where good error messages matter most.
- pest-parser/pest — PEG grammars written in a separate `.pest` file; choose it when you want a readable declarative grammar over hand-composed functions.
- lalrpop/lalrpop — an LR(1) parser generator; choose it for conventional language grammars that suit a generated bottom-up parser.
- Marwes/combine — another parser-combinator library with a different trait design; a direct alternative if nom's API does not fit.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015 | Early releases, macro-based combinators. |
| 4.0 | 2018 | Large API changes, still macro-heavy. |
| 5.0 | 2019-08 | Macros → functions; the first big migration break[^4]. |
| 6.0 | 2021-01 | Continued function-based cleanup. |
| 7.0 | 2021-08 | Remaining macros removed; module reorganization. |
| 8.0 | 2025 | `Parser`-trait redesign; `.parse()` method API, `tuple()` deprecated. |

## References

[^1]: nom README, "nom, eating data byte by byte". https://github.com/rust-bakery/nom
[^2]: crates.io — nom (reverse dependencies). https://crates.io/crates/nom/reverse_dependencies
[^3]: nom README, "Rust version requirements (MSRV)" — 8.0 series supports rustc 1.65+. https://github.com/rust-bakery/nom#rust-version-requirements-msrv
[^4]: nom upgrading notes / CHANGELOG. https://github.com/rust-bakery/nom/blob/main/CHANGELOG.md

## Tags

rust, parser, parser-combinators, binary-parsing, streaming, zero-copy, no-std, protocols, library
