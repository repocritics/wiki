# mrkkrp/megaparsec

> Industrial-strength monadic parser combinator library for Haskell — the Parsec fork that became the ecosystem default for parsing human-readable input.

[GitHub repo](https://github.com/mrkkrp/megaparsec) ·
[Hackage](https://hackage.haskell.org/package/megaparsec) ·
[License: BSD-2-Clause (FreeBSD)](https://github.com/mrkkrp/megaparsec/blob/master/LICENSE.md)

## Overview

Megaparsec is a parser combinator library for Haskell, started in 2015 by Mark Karpov as a fork of Parsec — itself descended from Daan Leijen's original 1999–2000 work[^1]. Parsec had stagnated; Megaparsec modernized it: typed and extensible error messages, multi-error reporting, error recovery mid-parse, indentation-sensitive parsing, and Attoparsec-style fast chunk primitives. It is now the conventional choice for parsing source code and configuration languages in Haskell — Idris, Dhall, hledger, hnix, and MMark all parse with it[^2].

The library's stated design goal is a balance between speed, flexibility, and parse error quality — and that balance is a real tradeoff, not marketing. Attoparsec is sometimes faster but has poor errors; Trifecta has pretty errors but a heavy dependency footprint (`lens`) and thin documentation; Earley safely handles context-free grammars but is slower and stateless. Megaparsec sits in the middle: parsers written carefully with its chunk primitives approach Attoparsec speed, while retaining errors good enough to show a compiler user[^3].

At 974 stars it looks modest by JavaScript standards, but for a Haskell library this is top-tier adoption — it is a Stackage LTS fixture and a transitive dependency of much of the ecosystem's tooling. Development is steady rather than fast: a single primary maintainer, a mature API, and pushes as recent as June 2026. The library is finished software in the good sense; expect bug fixes and GHC-compatibility bumps, not feature churn.

## Getting Started

```bash
cabal install --lib megaparsec parser-combinators
# or add to package.yaml / .cabal: megaparsec, parser-combinators
```

```haskell
{-# LANGUAGE OverloadedStrings #-}
import Data.Text (Text)
import Data.Void (Void)
import Text.Megaparsec
import Text.Megaparsec.Char
import qualified Text.Megaparsec.Char.Lexer as L

type Parser = Parsec Void Text

sc :: Parser ()                       -- space consumer
sc = L.space space1 (L.skipLineComment "--") empty

integer :: Parser Int
integer = L.lexeme sc L.decimal

pair :: Parser (Text, Int)
pair = do
  key <- L.lexeme sc (takeWhile1P (Just "letter") (/= '='))
  _   <- L.symbol sc "="
  val <- integer
  pure (key, val)

main :: IO ()
main = case parse (pair <* eof) "<input>" "port = 8080" of
  Left err -> putStr (errorBundlePretty err)
  Right kv -> print kv
```

`errorBundlePretty` renders the offending line with a caret — the error-message quality that motivates choosing Megaparsec in the first place.

## Architecture / How It Works

The core abstraction is **`MonadParsec`**, an MTL-style type class, with **`ParsecT e s m a`** as the canonical transformer instance (`e` = custom error component, `s` = input stream, `m` = underlying monad). Because `StateT`, `ReaderT`, `WriterT` etc. are `MonadParsec` instances, you can stack them in either order — `ParsecT` over `StateT` gives backtracking user state; `StateT` over `ParsecT` does not. This ordering choice is semantically significant and a classic source of subtle bugs[^3].

**Streams.** Input is abstracted behind the `Stream` type class, with instances for `String`, strict/lazy `ByteString`, and strict/lazy `Text`, plus support for custom token streams (e.g. tokens produced by an Alex lexer). The class was redesigned significantly in versions 6 and 7, which is why old blog posts often show code that no longer compiles.

**Chunk primitives.** `tokens`, `takeWhileP`, `takeWhile1P`, and `takeP` consume input as chunks of the underlying stream (a `Text` slice stays `Text`, no repacking). The README quantifies the gap: `tokens` is roughly 100× faster than matching character-by-character, `takeWhileP` roughly 150× faster than `many`-based equivalents[^3]. Performance is therefore not automatic — it depends on the author reaching for these primitives instead of naive combinators.

**Errors.** Parse errors are typed (`ParseError` with a custom component slot) and collected into a `ParseErrorBundle` for pretty-printing. Three unusual capabilities: `withRecovery` resumes parsing after a failure so multiple errors can be reported in one pass (compiler-style diagnostics); `observing` inspects a failure without aborting; `parseError` (since v8) raises an error at a position independent of the current offset[^4].

**Lexing.** `Text.Megaparsec.Char.Lexer` provides the space-consumer convention (`lexeme`/`symbol` wrap tokens and eat trailing whitespace) plus indentation combinators (`indentBlock`, `nonIndented`) — the piece that makes Python-style layout parsing tractable. Expression parsing (`makeExprParser`) and permutation parsers were split into the separate `parser-combinators` package in v7[^5].

## Production Notes

- **Backtracking is explicit.** Like Parsec, a branch that consumes input and then fails commits — `<|>` will not try the alternative. You must wrap with `try`, and overusing `try` both destroys error quality and can degrade performance to exponential in pathological grammars. Placement of `try`, `label`/`<?>`, and `hidden` is the actual craft of writing a Megaparsec parser.
- **Naive parsers are slow.** A parser built from `many (satisfy ...)` on `Text` allocates per character. Rewriting hot paths with `takeWhileP`/`tokens` is routinely a 10–100× win and is required to approach the Attoparsec-comparable benchmark numbers[^3].
- **Left recursion is unsupported.** This is recursive descent; left-recursive grammar rules loop forever. Use `makeExprParser` from `parser-combinators` for operator precedence, or refactor the grammar.
- **Type parameters leak everywhere.** `ParsecT e s m a` means every helper signature carries your error and stream types. Standard practice is a project-wide alias (`type Parser = Parsec Void Text`); introducing a custom error component later touches every signature and requires a `ShowErrorComponent` instance.
- **Transformer overhead is real.** `ParsecT`'s CPS core plus a monad stack costs relative to Attoparsec's specialized design; the third-party `faster-megaparsec` package exists precisely to claw some of it back[^6]. For raw throughput on machine-generated data (network protocols, large binary/log ingestion), Attoparsec or flatparse remain the better tools.
- **Major-version migrations were disruptive.** v6 (2017) redesigned `Stream` and the module layout; v7 (2018) reworked errors into `ParseErrorBundle` and evicted `parser-combinators`; v8 (2020) changed error-position semantics[^4][^5]. The API has been stable through the 9.x line, but code and tutorials predating v7 need translation.
- **Streaming is not incremental.** Unlike Attoparsec, Megaparsec has no resumable/partial-input mode; lazy `Text`/`ByteString` streams help with memory but the parse still expects the full input to be ultimately available.

## When to Use / When Not

**Use when:**
- Parsing source code, DSLs, or config formats where humans read the error messages.
- You need indentation-sensitive syntax, custom error types, or multi-error reporting with recovery.
- You want a monad-transformer parser that composes with `StateT`/`ReaderT` effects.
- You are choosing the Haskell-ecosystem default and want tutorials, Stackage stability, and real-world reference users (Dhall, hledger, Idris).

**Avoid when:**
- Raw throughput on large machine-generated input is the priority — Attoparsec or flatparse win, and error quality does not matter there.
- You need incremental/resumable parsing of network streams — Attoparsec's partial results have no Megaparsec equivalent.
- Your grammar is genuinely context-free and safety from ambiguity matters more than speed — Earley handles left recursion and ambiguity that recursive descent cannot.
- You are not in Haskell; this design has idiomatic ports elsewhere (e.g. nom in Rust, parsy in Python) rather than bindings.

## Alternatives

- haskell/attoparsec — use instead when parsing large volumes of machine-generated data where speed and incremental input beat error quality.
- haskell/parsec — use instead only for legacy codebases or minimal-dependency constraints; Megaparsec is its maintained successor in practice.
- ekmett/trifecta — use instead if you want clang-style ANSI diagnostics and already live in the `lens` ecosystem.
- ollef/Earley — use instead for context-free grammars needing left recursion or ambiguity handling, at a speed cost.
- AndrasKovacs/flatparse — use instead for maximum-throughput non-resumable parsing where you accept a lower-level API.

## History

| Version | Date | Notes |
|---------|------|-------|
| 4.0.0 | 2015-09 | Initial release as a Parsec fork; announced on haskell-cafe[^1]. |
| 5.0 | 2016 | Custom error components; typed error data. |
| 6.0 | 2017 | `Stream` redesign; Attoparsec-style chunk primitives (`takeWhileP`, `tokens`); major speedup[^7]. |
| 7.0 | 2018 | `ParseErrorBundle`; expression/permutation parsers split into `parser-combinators`[^5]. |
| 8.0 | 2020 | Error positions decoupled from stream offset; easier multi-error reporting; `parseError`[^4]. |
| 9.x | 2021– | Stable maintenance line; incremental releases, GHC compatibility. Active as of June 2026. |

## References

[^1]: Original Megaparsec 4.0.0 announcement, haskell-cafe — September 2015. https://mail.haskell.org/pipermail/haskell-cafe/2015-September/121530.html
[^2]: Megaparsec README, "Prominent projects that use Megaparsec". https://github.com/mrkkrp/megaparsec#prominent-projects-that-use-megaparsec
[^3]: Megaparsec README — performance section and benchmark table (parsers-bench). https://github.com/mrkkrp/megaparsec#performance
[^4]: Mark Karpov, "Megaparsec 8". https://markkarpov.com/post/megaparsec-8.html
[^5]: Mark Karpov, "Megaparsec 7". https://markkarpov.com/post/megaparsec-7.html
[^6]: faster-megaparsec on Hackage. https://hackage.haskell.org/package/faster-megaparsec
[^7]: Mark Karpov, "A major upgrade to Megaparsec: more speed, more power". https://markkarpov.com/post/megaparsec-more-speed-more-power.html

## Tags

haskell, parsing, parser-combinators, compiler-frontend, dsl, error-reporting, lexing, monad-transformers, library
