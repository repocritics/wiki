# zesterer/chumsky

> Rust parser-combinator library built around error recovery and zero-copy parsing — the "compiler-frontend-grade diagnostics" option among Rust parsers.

[GitHub repo (archived mirror)](https://github.com/zesterer/chumsky) ·
[Canonical repo (Codeberg)](https://codeberg.org/zesterer/chumsky) ·
[License: MIT](https://github.com/zesterer/chumsky/blob/main/LICENSE)

## Overview

Chumsky is a parser-combinator library for Rust by Joshua Barretto (zesterer), first released in July 2021[^2]. Parsers are built by composing small typed combinators (`just`, `choice`, `recursive`, `delimited_by`, …) into a recursive descent parser capable of handling PEGs, plus context-sensitive constructs (Rust raw strings, Python-style indentation) via dedicated combinators[^1]. Its distinguishing bet, relative to nom and winnow, is that error *quality* is the hard part of parsing: rich spans, pattern labelling, and error-recovery strategies that produce a partial AST plus multiple diagnostics from broken input are first-class, which is why it is most at home in compiler and DSL frontends (often paired with the same author's `ariadne` for rendering[^4]).

The project's history has a versioning trap. A ground-up "zero-copy" rewrite was developed as `1.0.0-alpha.0` through `alpha.8` (2023–2025), then the 1.0 label was abandoned and the rewrite shipped as **0.10.0** in March 2025[^2]. On crates.io, `1.0.0-alpha.*` is therefore *older* than `0.10+` despite sorting higher. Current stable is 0.13.0 (May 2026); the crate has ~25M downloads[^2]. At 4.5k GitHub stars it sits in the top tier of Rust parsing libraries, though well behind nom's installed base.

In 2026 the project left GitHub for Codeberg; the GitHub repo is now an archived mirror (last push 2026-03-27) and issues/PRs happen on Codeberg[^5]. The maintainer is explicit that the codebase is human-authored and asks contributors not to submit LLM-generated changes[^1] — relevant if your organization's contribution workflow assumes AI-assisted patches are acceptable.

## Getting Started

```bash
cargo add chumsky
```

A minimal recursive parser (trimmed from the repo's Brainfuck example[^1]):

```rust
use chumsky::prelude::*;

#[derive(Clone, Debug)]
enum Instr { Incr, Decr, Loop(Vec<Instr>) }

fn parser<'a>() -> impl Parser<'a, &'a str, Vec<Instr>> {
    recursive(|instr| choice((
        just('+').to(Instr::Incr),
        just('-').to(Instr::Decr),
        instr.delimited_by(just('['), just(']')).map(Instr::Loop),
    ))
    .repeated()
    .collect())
}

fn main() {
    println!("{:?}", parser().parse("+-[+[-]]").into_result());
}
```

The docs.rs guide is unusually complete — combinator index, theory, and a tutorial building a full interpreter for a small language[^3].

## Architecture / How It Works

The `Parser<'a, I, O, E>` trait is generic over input, output, error, and extra state, with a lifetime tying outputs to the input — this is what enables **zero-copy parsing**: outputs can be `&str`/slice references into the source rather than owned allocations. The 0.10+ design leans heavily on generic associated types (GATs, stable since Rust 1.65) to implement an internal "many modes" optimiser[^1]: the same parser definition compiles into distinct specialized code paths, notably a **check-only mode** that validates input without constructing any output — in the README's own JSON benchmark, check mode runs ~1.5× faster than full parsing[^1].

Execution is recursive descent. That means left-recursive grammars loop by default; chumsky offers opt-in left-recursion support via a `memoization` feature rather than grammar transformation. Context-sensitivity is handled by combinators that thread parsed values into subsequent parsers, letting it go beyond context-free grammars. Other opt-in features: `pratt` (operator precedence parsing), `regex`, `extension` (write your own first-class combinators), and `no_std` operation for embedded targets. The default-enabled `stacker` feature spills the recursion stack to the heap to avoid overflows on deeply nested input[^1].

The cost of all this genericity is paid at compile time: a chumsky parser is one enormous nested type, and both build times and type-error messages scale with parser complexity (see Production Notes).

## Production Notes

- **The version-number trap is real.** `cargo add chumsky@1.0.0-alpha.8` gets you a *dead branch* from January 2025. Anything current is `0.10+`; treat the 1.0-alpha line as abandoned[^2]. Older tutorials and StackOverflow answers targeting 0.9 do not compile against 0.10+ — the rewrite changed the `Parser` trait signature (added lifetime, `extra` param) and the error API, so a 0.9 → 0.10 upgrade is a rewrite of your parser code, not a bump.
- **Compile-time and diagnostic blowup.** Deeply composed parsers produce huge monomorphized types; a type mismatch mid-chain can emit multi-screen compiler errors. The standard mitigation is `.boxed()` at module boundaries, trading a virtual call for type erasure and sane build times.
- **Error richness costs throughput.** Parsing with `Rich` errors and spans is measurably slower than with cheap error types or check-only mode. The README's JSON benchmark places chumsky between winnow and nom for full parsing, and ahead of both in check-only mode[^1] — but it is one self-published benchmark on one grammar; benchmark your own.
- **Deep recursion.** Without the default `stacker` feature (e.g. some `no_std` builds), pathological nesting in untrusted input can overflow the stack — a denial-of-service consideration for network-facing parsers.
- **Semver caveat:** APIs behind the `unstable` feature are explicitly excluded from semver guarantees[^1].
- **Project home moved.** File issues on Codeberg, not the archived GitHub mirror; stale GitHub links persist across docs and blog posts. Releases continue post-move (0.13.0, 2026-05)[^2], so the move signals platform preference, not abandonment.

## When to Use / When Not

**Use when:**

- You are building a compiler, interpreter, or DSL frontend where multi-error reporting and recovery from broken input matter.
- Your syntax is still evolving and you want to iterate on grammar in typed Rust code rather than a separate grammar file.
- You need context-sensitive constructs (indentation, raw strings) that pure CFG tools handle poorly.
- You want zero-copy outputs borrowing from the source text.

**Avoid when:**

- You are parsing high-volume binary protocols where throughput dominates and diagnostics are irrelevant — winnow/nom are leaner fits.
- You need a grammar as a reviewable artifact separate from code (pest, LALRPOP).
- Compile time is already a pain point; heavy combinator nesting makes it worse.
- You need left recursion without opting into memoization, or LR-class grammar guarantees.

## Alternatives

- winnow-rs/winnow — use instead when you want an imperative-style, throughput-first parser for binary or simple text formats.
- rust-bakery/nom — use instead when you want the largest ecosystem of examples and battle-tested binary-format parsers.
- pest-parser/pest — use instead when you want the grammar in a standalone PEG file rather than Rust combinators.
- lalrpop/lalrpop — use instead when you want an LR(1) generator with grammar-level ambiguity checking for a classical compiler pipeline.
- maciejhirsz/logos — not a parser but the de facto lexer companion; commonly paired with chumsky (the repo ships an integration example).

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2021-07 | Initial release[^2]. |
| 0.8 | 2022-02 | Last major release before the 0.9 API consolidation. |
| 0.9 | 2023-02 | Final feature line of the original owned-output API. |
| 1.0.0-alpha.0–.8 | 2023-03 → 2025-01 | Zero-copy/GATs rewrite; label later abandoned[^2]. |
| 0.10.0 | 2025-03 | The rewrite ships as 0.10 instead of 1.0; breaking API. |
| 0.13.0 | 2026-05 | Current stable; first release after the Codeberg move[^2]. |

## References

[^1]: chumsky README (features, classification, benchmark, provenance statement). https://github.com/zesterer/chumsky
[^2]: crates.io version history and download counts for `chumsky`. https://crates.io/crates/chumsky/versions
[^3]: chumsky guide on docs.rs. https://docs.rs/chumsky/latest/chumsky/guide/
[^4]: ariadne — diagnostic rendering crate by the same author. https://github.com/zesterer/ariadne
[^5]: Canonical repository on Codeberg. https://codeberg.org/zesterer/chumsky

## Tags

rust, parsing, parser-combinators, error-recovery, peg, recursive-descent, zero-copy, no-std, compilers, dsl
