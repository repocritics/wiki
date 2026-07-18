# dashbitco/nimble_parsec

> Parser combinators for Elixir that compile to raw binary-matching function clauses — combinator ergonomics at hand-written-parser speed.

[GitHub repo](https://github.com/dashbitco/nimble_parsec) ·
[Docs](https://hexdocs.pm/nimble_parsec) ·
[License: Apache-2.0](https://github.com/dashbitco/nimble_parsec/blob/master/LICENSE)

## Overview

NimbleParsec is a text-parsing library for Elixir, started at Plataformatec in 2018 and maintained under Dashbit since 2020[^1], part of the "Nimble" family of deliberately small libraries (NimbleCSV, NimbleOptions, NimblePool). Unlike classic parser-combinator libraries, combinators here are not functions interpreted at runtime — they are data structures that the `defparsec/3` macro compiles into private function clauses using Erlang binary pattern matching[^2]. The BEAM's binary-matching optimizations then apply directly, which is why the README's claim of an order-of-magnitude advantage over interpreted combinator libraries is credible rather than marketing[^2].

The defining tradeoff is compile time versus runtime. Every combinator is inlined at each use site, so large or heavily-reused grammars inflate module compile times and memory; the escape hatch is `defcombinatorp` + `parsec/1`, which references compiled combinators instead of re-inlining them[^2]. The second tradeoff is scope: text parsing only — the README itself redirects low-level binary-protocol work to Elixir's bitstring syntax[^2].

At 877 stars it is small by absolute numbers but load-bearing in the Elixir ecosystem: Makeup (the syntax highlighter behind ExDoc) and many format parsers on Hex build on it. The last feature release was v1.4.0 (2023); the 1.4.x patches (Jan 2025) only silence Elixir 1.18 warnings[^3]. Read that as feature-complete maintenance, not abandonment — the API has been stable since 1.0 (2020).

## Getting Started

```elixir
# mix.exs
def deps do
  [{:nimble_parsec, "~> 1.0"}]
end
```

```elixir
defmodule MyParser do
  import NimbleParsec

  date =
    integer(4)
    |> ignore(string("-"))
    |> integer(2)
    |> ignore(string("-"))
    |> integer(2)

  defparsec :date, date
end

MyParser.date("2010-04-17")
#=> {:ok, [2010, 4, 17], "", %{}, {1, 0}, 10}
```

The ok-tuple carries the accumulated results, the unconsumed rest, a user-managed context map, `{line, line_start_offset}`, and the byte offset.

## Architecture / How It Works

Combinators (`string/2`, `integer/2`, `ascii_string/3`, `choice/2`, `repeat/2`, `lookahead/2`, …) build an instruction list — plain data, no macros involved, so composition is ordinary function piping. The only macros are `defparsec/3`, `defparsecp/3`, `defcombinator/3`, and `defcombinatorp/3`, which hand the instruction list to the compiler[^2].

The compiler unrolls the grammar into a chain of private functions (`name__0`, `name__1`, …), each a set of clauses that pattern-match directly on the input binary with integer guards. Passing `debug: true` to `defparsec/3` prints the generated clauses — the output is essentially the recursive-descent parser you would have written by hand[^2]. Consequences:

- **Semantics are PEG-like.** `choice/2` is ordered: alternatives are tried top-down and the first match wins. No longest-match or ambiguity handling; bugs from mis-ordered alternatives are on you.
- **Recursion** requires the indirection of `parsec(:name)` pointing at a `defcombinatorp`, since inlining a recursive grammar would not terminate.
- **Traversals** (`post_traverse/3`, `pre_traverse/3`, `map`, `reduce`, `tag`, `unwrap_and_tag`) transform the accumulator during the parse, so most users build their AST inline rather than post-processing token lists.
- **Generator mode** (v1.2.0, 2021) runs combinators in reverse to produce strings a parser accepts — usable for property-based testing[^3].
- **Zero runtime dependency.** Generated clauses reference nothing from NimbleParsec, and `mix nimble_parsec.compile` emits a standalone source file, so libraries can vendor a parser without imposing the dependency[^2].

## Production Notes

**Compile-time blowup is the real footgun.** Reusing a combinator variable in several places compiles it once per use site; a moderately sized grammar written naively can take long enough to compile that people file it as a bug. The fixes are documented but easy to miss: wrap shared pieces in `defcombinatorp` + `parsec/1`, and since v1.1.0 split grammars across modules with remote `parsec` for parallel compilation[^3]. `inline: true` trades even more compile time for runtime speed — benchmark before enabling.

**Error messages are minimal.** Failures return `"expected <description>"` plus line/byte offset. For a deep grammar the auto-generated description is an unreadable union of alternatives; production parsers need `label/2` applied liberally. If diagnostics quality is a primary requirement (e.g. a compiler front-end), this is the library's weakest area.

**No streaming.** Input must be one in-memory binary. Multi-gigabyte files or incremental network parsing are out of scope.

**Backtracking costs.** `choice/2` rewinds input on failure, so alternatives sharing long prefixes re-parse them per branch; `lookahead/2` / `lookahead_not/2` prune cheaply. Pathological nesting of `choice` + `repeat` can degrade sharply.

**Upgrade history is gentle.** The only significant breaks were pre-1.0: v0.5.0 (2018) removed `repeat_until/3` and renamed `traverse` to `post_traverse`[^3]. v1.3.0 (2023) deprecated the two-element return from quoted traversals. Code written against 1.0 in 2020 generally still compiles; v1.4.0 raised the floor to Elixir 1.12.

## When to Use / When Not

**Use when:**
- You are parsing a text format (dates, log lines, config DSLs, Markdown-ish markup) in Elixir and want hand-written-parser performance without writing the clauses yourself.
- You are shipping a library and want parsing without adding a runtime dependency (`mix nimble_parsec.compile`).
- Your grammar is stable — the compile-time cost is paid rarely.

**Avoid when:**
- You need rich, user-facing parse errors with recovery; the error model is too thin without heavy `label/2` work.
- You are parsing binary protocols — Elixir bitstring syntax is the right tool, per the README itself[^2].
- You need streaming/incremental parsing of unbounded input.
- Your grammar is huge and changes constantly; recompile times will hurt the edit loop.

## Alternatives

- erlang/otp (yecc + leex) — use the OTP-bundled LALR generator when you want a classic tokenizer/grammar split and grammar files over Elixir code.
- bitwalker/combine — runtime-interpreted combinators; simpler to step through, slower, and effectively unmaintained.
- mmower/ergo — runtime combinators focused on debuggability and telemetry of the parse; use when understanding parser behavior matters more than speed.
- princemaple/abnf_parsec — feed it an RFC-style ABNF grammar directly (itself built on NimbleParsec) instead of hand-porting the grammar.
- Hand-written binary matching — when you need both maximum speed and precise error reporting, the generated-code approach shows you the pattern to copy.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2018-03-02 | First release, at Plataformatec[^3]. |
| 0.3.0 | 2018-04-08 | `tag`/`unwrap`; `mix nimble_parsec.compile` for dependency-free parsers[^3]. |
| 0.5.0 | 2018-12-12 | `pre_traverse`, combinator `lookahead`/`lookahead_not`, `eos`; removed `repeat_until`[^3]. |
| 1.0.0 | 2020-09-25 | API stabilized, now under Dashbit[^3]. |
| 1.1.0 | 2020-10-02 | `defcombinator` + remote `parsec` — split grammars across modules for parallel compilation[^3]. |
| 1.2.0 | 2021-11-07 | Generator support (produce strings matching a combinator)[^3]. |
| 1.3.0 | 2023-03-27 | Quoted-traversal return-shape deprecation; lookahead fixes[^3]. |
| 1.4.0 | 2023-11-08 | Requires Elixir 1.12[^3]. |
| 1.4.2 | 2025-01-21 | Elixir 1.18 warning cleanup — maintenance-only cadence[^3]. |

## References

[^1]: LICENSE — Copyright 2018 Plataformatec, Copyright 2020 Dashbit. https://github.com/dashbitco/nimble_parsec/blob/master/LICENSE
[^2]: NimbleParsec README — design goals, generated-clause example, performance considerations. https://github.com/dashbitco/nimble_parsec#readme
[^3]: NimbleParsec CHANGELOG. https://github.com/dashbitco/nimble_parsec/blob/master/CHANGELOG.md

## Tags

elixir, erlang-vm, parser-combinator, parsing, text-processing, compile-time-codegen, peg, library, dashbit, beam
