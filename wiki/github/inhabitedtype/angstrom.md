# inhabitedtype/angstrom

> OCaml parser combinators built for network protocols: backtracking by
> default, incremental input, and a zero-copy unbuffered mode — a deliberate
> break from the Parsec lineage.

[GitHub repo](https://github.com/inhabitedtype/angstrom) ·
[License: BSD-3-Clause](https://github.com/inhabitedtype/angstrom/blob/master/LICENSE)

## Overview

Angstrom is a parser-combinator library for OCaml, started in 2015 by Spiros
Eliopoulos (Inhabited Type LLC) as a port of Haskell's attoparsec[^1]. The
continuation-passing core survives from the original, but the library was
re-engineered around OCaml's ecosystem: bigstring buffers, monadic concurrency
(Lwt and Async adapters ship in-repo), and an unbuffered interface that
supports zero-copy IO. Its target domain is explicitly network protocols and
serialization formats, not programming-language frontends — the source tree
ships RFC 2616 (HTTP) and RFC 7159 (JSON) parsers as examples[^2].

The defining design choice is **backtracking by default with unbounded
lookahead**. Where Parsec derivatives (mparser, planck, opal) require an
explicit `try` to backtrack and model input as a lazy character stream,
Angstrom's `<|>` always retries the alternative from the same position, and
input is a buffer the parser can iterate over in tight loops. That buys
predictable composition and fast bulk combinators (`take_while`,
`skip_while`), and costs error diagnostics: Angstrom does not report line
numbers, a gap its own README concedes[^2].

Within the OCaml world Angstrom is the de facto standard combinator library
for binary and wire-format parsing — it sits under httpaf/h2 and the uri
library, among others[^6][^7]. At 705 stars and 74 forks it is small by GitHub
standards but large for OCaml. Development is dormant: the last release and
last push were both 0.16.1 on 2024-09-12, after a 2020–2023 gap between 0.15.0
and 0.16.0. Treat it as finished-and-stable rather than actively evolving; the
13 open issues move slowly.

## Getting Started

```bash
opam install angstrom
```

The customary arithmetic-expression parser, evaluating as it parses[^2]:

```ocaml
open Angstrom

let parens p = char '(' *> p <* char ')'
let add = char '+' *> return (+)
let integer =
  take_while1 (function '0' .. '9' -> true | _ -> false) >>| int_of_string

let chainl1 e op =
  let rec go acc =
    (lift2 (fun f x -> f acc x) op e >>= go) <|> return acc in
  e >>= fun init -> go init

let expr : int t =
  fix (fun expr ->
    let factor = parens expr <|> integer in
    chainl1 factor add)

let eval (str : string) : int =
  match parse_string ~consume:All expr str with
  | Ok v -> v
  | Error msg -> failwith msg
```

`~consume` is required since 0.14.0: `All` fails on trailing input, `Prefix`
(the old implicit behavior) silently accepts it[^3].

## Architecture / How It Works

The core is a continuation-passing-style parser inherited from attoparsec: a
parser is a function over the current input state, an "is more input coming"
flag, and failure/success continuations[^1]. Combinator composition builds one
big closure chain; there is no intermediate AST or grammar object.

- **Backtracking and buffering.** Because `<|>` may rewind, consumed input
  cannot be discarded until the parser passes a `commit`. `commit` is the
  pressure-release valve: it promises no backtracking past the current
  position, letting the buffer be reclaimed. Long-running streaming parsers
  without periodic `commit`s retain the entire input.
- **Buffered vs. Unbuffered.** The `Buffered` interface accepts `` `String ``
  or `` `Bigstring `` chunks and manages an internal buffer. The `Unbuffered`
  interface returns `Partial` states carrying a committed-byte count and reads
  directly from caller-owned bigstrings — this is the zero-copy path, and the
  reason httpaf can parse straight out of socket buffers[^6].
- **Bulk combinators.** `take_while`/`skip_while`/`string` iterate over the
  buffer in tight loops rather than character-at-a-time through the monad —
  the main performance differentiator from lazy-stream designs[^2].
- **Concurrency adapters.** `angstrom-async`, `angstrom-lwt-unix`, and
  `angstrom-unix` are separate opam packages built from the same repo, keeping
  the core dependency-light.
- **Companion library.** Faraday, by the same author, is the serialization
  mirror image; httpaf is built on the pair[^5][^6].

## Production Notes

- **Error messages are the weak point.** Failures surface as a stack of
  context marks plus a terse message — no line/column, no expected-token sets.
  `<?>` labels help, but if you are building a compiler or anything where
  humans read parse errors, Angstrom is the wrong tool.
- **`commit` discipline is not optional in streaming.** Omitting it turns an
  incremental parser into an accumulate-everything parser. The interaction of
  `<|>` with commit points is also subtle — a bug where `<|>` did not respect
  commit points survived until 0.10.0[^4].
- **Backtracking hides errors.** Since `<|>` always retries, a deep failure in
  the first branch is discarded and the reported error comes from the last
  alternative — often far from the real problem.
- **Write hot paths applicatively.** Monadic `>>=` allocates closures per
  step; the `*>`/`<*`/`lift2` style and the `take_while` family are what keep
  Angstrom fast. Naive char-by-char monadic parsers forfeit the performance
  argument.
- **Breaking changes were real but are long past:** `intn` became `get_intn`
  in 0.10.0, `~consume` became mandatory in 0.14.0[^3][^4]. Anything written
  against 0.14+ still compiles today.
- **Dormancy risk.** No repository activity since September 2024. The library
  is small and stable enough that this is tolerable, but budget for vendoring
  or forking if you hit an edge case; upstream response times are unbounded.
- **js_of_ocaml works**, including a trampolined `fix` (0.13.0) to avoid stack
  overflow on deeply recursive parsers compiled to JavaScript[^3].

## When to Use / When Not

**Use when:**
- Parsing wire protocols or binary/serialization formats in OCaml, especially
  under Lwt or Async.
- You need incremental parsing of data that arrives in chunks, or zero-copy
  parsing from network buffers.
- You want attoparsec ergonomics — backtracking without `try`, fast bulk
  scanners — in OCaml.

**Avoid when:**
- Humans will read your parse errors (config files, DSLs, compilers) — use a
  Parsec-style library or a parser generator with real diagnostics.
- Your grammar is a programming language: an LR generator like Menhir checks
  the grammar for ambiguity and produces better errors.
- You need an actively maintained dependency with fast upstream turnaround.

## Alternatives

- cakeplus/mparser — Parsec-style with line/column error reporting; use it
  when diagnostics matter more than throughput.
- Menhir (Inria GitLab, not on GitHub) — LR(1) parser generator; use it for
  programming-language grammars where ambiguity checking and error recovery
  count.
- pyrocat101/opal — a single-file, dependency-free combinator kit; use it for
  scripts and toys where installing a library is overkill.
- bos/attoparsec — the Haskell original Angstrom was ported from; use it when
  you are in Haskell.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.7.0 | 2017-09-16 | Dropped cstruct and ocplib-endian deps; removed string-backed internal buffer[^3]. |
| 0.10.0 | 2018-06-10 | `intn` → `get_intn` rename; fixed `<|>` ignoring commit points[^4]. |
| 0.11.1 | 2019-03-03 | Build ported to dune[^3]. |
| 0.13.0 | 2020-03-11 | Trampolined `fix` for js_of_ocaml[^3]. |
| 0.14.0 | 2020-04-25 | `parse_string`/`parse_bigstring` require `~consume`[^3]. |
| 0.15.0 | 2020-09-29 | let-operator / ppx_let syntax support[^3]. |
| 0.16.0 | 2023-11-22 | First release after a three-year gap. |
| 0.16.1 | 2024-09-12 | Latest release; last repository activity to date. |

## References

[^1]: Angstrom README, "Acknowledgements" — the library began as a port of attoparsec. https://github.com/inhabitedtype/angstrom#acknowledgements
[^2]: Angstrom README — feature comparison table, examples, and design rationale. https://github.com/inhabitedtype/angstrom#readme
[^3]: Angstrom release notes, 0.7.0–0.15.0. https://github.com/inhabitedtype/angstrom/releases
[^4]: Angstrom 0.10.0 release notes — `get_intn` rename and `<|>` commit-point fix (#146). https://github.com/inhabitedtype/angstrom/releases/tag/0.10.0
[^5]: Faraday — companion serialization library. https://github.com/inhabitedtype/faraday
[^6]: httpaf — high-performance HTTP implementation built on Angstrom and Faraday. https://github.com/inhabitedtype/httpaf
[^7]: ocaml-uri — RFC 3986 URI library; its 4.x parser is written with Angstrom. https://github.com/mirage/ocaml-uri

## Tags

ocaml, parser-combinators, parsing, incremental-parsing, streaming, zero-copy, network-protocols, backtracking, library
