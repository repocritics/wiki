# dtolnay/indoc

> Compile-time un-indentation for Rust multiline string literals, so indented source stays readable without leaking leading whitespace into the value.

[GitHub repo](https://github.com/dtolnay/indoc) ·
[docs.rs](https://docs.rs/indoc) ·
[crates.io](https://crates.io/crates/indoc) ·
License: Apache-2.0 OR MIT (dual)

## Overview

`indoc` is a small procedural macro crate by David Tolnay that solves one narrow, persistent annoyance: a multiline string literal written inside indented Rust code carries that indentation into the string. Without help, embedding a Python snippet, a SQL query, or a help-text block inside a function means either breaking the visual flow (flushing the literal to column zero) or shipping unwanted leading spaces. `indoc!{ ... }` takes the literal, finds the common leading-space prefix across its non-blank lines, and strips it at compile time so the leftmost non-space character lands in column one[^1].

The audience is anyone embedding formatted text in Rust: CLI help output, test fixtures, code generators, templating, doctests, and inline DSLs. It is one of the most widely transitively depended-upon crates in the ecosystem, largely because it appears in the dependency trees of testing and codegen tooling rather than being reached for directly by application authors.

The defining tension is that `indoc!` operates entirely at compile time on a *literal* token. That is what makes it zero-cost and usable in `const`/`static` position, but it also means it cannot touch a runtime `String` or a value behind a variable. For that case the same author ships the sibling `unindent` crate, which performs the identical algorithm on ordinary runtime strings.

## Getting Started

```toml
[dependencies]
indoc = "2"
```

```rust
use indoc::indoc;

let s = indoc! {"
    def hello():
        print('Hello, world!')

    hello()
"};
// s == "def hello():\n    print('Hello, world!')\n\nhello()\n"
```

The macro accepts raw string literals (`r#"..."#`) and byte string literals (`b"..."`) unchanged, so escaping rules are yours to choose. Blank lines are preserved; only the common leading-space run is removed.

## Architecture / How It Works

The algorithm is deliberately simple and fully specified in the README[^1]:

1. Count leading spaces on each line, ignoring the first line and any line that is empty or spaces-only.
2. Take the minimum across the remaining lines — that is the common indent.
3. If the string begins with a newline (the first line is empty), drop that first line.
4. Strip the computed number of spaces from every line.

Because this runs as a proc macro over the token stream, `indoc!` sees the *decoded* contents of the literal, not the raw source bytes, and it emits a new string literal token. The result is a plain `&'static str` (or `&'static [u8]` for byte strings) with no runtime work and no allocation, which is why it composes with `const` and `static` declarations.

Beyond the core macro, the crate exports formatting wrappers that pair `indoc!` with the standard macros so the format template itself can be indented[^2]:

- `formatdoc!` = `format!(indoc!(...), ...)`
- `printdoc!` / `eprintdoc!` = `print!` / `eprint!` over an un-indented template
- `writedoc!` = `write!(dest, indoc!(...), ...)`
- `concatdoc!` = `concat!(...)` with each string literal wrapped in `indoc!`, useful for stitching `env!`/`concat!` fragments into one indented block

The runtime counterpart lives in the `unindent` crate (`unindent(&str) -> String`, `unindent_bytes(&[u8]) -> Vec<u8>`), which shares the algorithm for strings that are not statically known.

Historically the crate's most interesting implementation detail was version 1.0's removal of `proc-macro-hack`. Before Rust 1.45 (2020) stabilized procedural macros in expression position, `indoc!` relied on Tolnay's `proc-macro-hack` shim to be callable inside an expression on stable Rust; 1.0 dropped that dependency once the language caught up[^3].

## Production Notes

- **Spaces only.** The algorithm counts *leading spaces*. Tab-indented literals are not un-indented the way tab-indented source suggests, and mixing tabs and spaces in the embedded block produces surprising output. Keep embedded literals space-indented.
- **Literal-only.** `indoc!` requires a string *literal* argument. It cannot un-indent a `String`, a `const` reference, or a value passed through a function. Reach for `unindent` at runtime instead — it allocates, unlike the macro.
- **The trailing newline is real.** A block written as `indoc!{"\n ...text\n"}` keeps its final `\n` because the closing line is empty-after-strip, not removed. Code comparing against a literal without a trailing newline (a common test-fixture mistake) will fail; strip explicitly if you need it gone.
- **Compile-time cost is negligible.** The crate is small and dependency-light, so it adds little to build graphs — a meaningful property given how often it lands in test/codegen dependency trees. It is a proc-macro dependency, so it still forces a proc-macro compilation stage on projects that had none.
- **Stability.** This is mature, low-churn infrastructure. Releases are infrequent and driven by ecosystem maintenance rather than feature growth; the API has been effectively frozen across the 2.x line. "Actively maintained" here means promptly patched, not evolving — appropriate for a utility of this scope.

## When to Use / When Not

**Use when:**
- You embed multiline text (help output, SQL, test fixtures, generated code, DSLs) and want the source indentation to match the surrounding block without polluting the value.
- You need the result in `const`/`static` or otherwise zero-allocation position.
- You want the formatting macros (`formatdoc!`, `writedoc!`) to accept an indented template.

**Avoid when:**
- The text is only known at runtime — use `unindent` (still by the same author) instead.
- Your literals are tab-indented and you are unwilling to convert to spaces.
- You want a single trailing-newline-free result without thinking about it; the newline semantics need one moment of attention.

## Alternatives

- dtolnay/unindent — the runtime sibling from the same repo; use it when the string is a `String`/`&str` value rather than a literal.
- mgeisler/textwrap — its `dedent()` and `indent()` operate on runtime strings and pair with wrapping/fill utilities; use when you also need width-aware wrapping.
- rodrimati1992/const_format — compile-time string formatting and concatenation; use when the need is `const` interpolation rather than de-indentation specifically.
- Plain `std` raw string literals flushed to column zero — use when a single block does not justify a dependency and you can tolerate the broken visual indentation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2016 | Initial release; proc-macro-hack era, usable on stable Rust via the expression-position shim[^3]. |
| 1.0 | 2021 | Dropped `proc-macro-hack` after Rust 1.45 stabilized proc macros in expression position; added the `*doc!` formatting family[^3]. |
| 2.0 | 2023 | Simplification/rewrite of the 1.x line; current major series (`indoc = "2"`). |

## References

[^1]: indoc README, "Explanation" — un-indentation rules. https://github.com/dtolnay/indoc#explanation
[^2]: indoc README, "Formatting macros" — `formatdoc!`, `printdoc!`, `eprintdoc!`, `writedoc!`, `concatdoc!`. https://github.com/dtolnay/indoc#formatting-macros
[^3]: Rust 1.45.0 release notes — procedural macros in expression, pattern, and statement position stabilized. https://blog.rust-lang.org/2020/07/16/Rust-1.45.0.html

## Tags

rust, procedural-macro, string-literals, compile-time, indentation, formatting, dtolnay, developer-tooling, text-processing, const
