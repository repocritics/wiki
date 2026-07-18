# snoyberg/conduit

> Haskell's workhorse streaming-data library — constant-memory pipelines with
> deterministic resource cleanup, at the cost of learning one more abstraction.

[GitHub repo](https://github.com/snoyberg/conduit) ·
[Hackage](https://hackage.haskell.org/package/conduit) ·
[License: MIT](https://github.com/snoyberg/conduit/blob/master/conduit/LICENSE)

## Overview

Conduit is a streaming-data framework for Haskell, created by Michael Snoyman
in December 2011 as a successor to the enumerator/iteratee style of
incremental IO that dominated early-2010s Haskell[^1]. It standardizes one
interface — `ConduitT i o m r` — for producing, transforming, and consuming
streams, so that a file reader, an HTTP response body, and a CSV parser all
compose with the same `.|` operator. Its selling points are constant memory
usage over arbitrarily large inputs, interleaved effects (each element flows
through the whole pipeline before the next is demanded), and deterministic
resource usage via the companion `resourcet` package[^2].

The defining tension: conduit is one of three-plus competing streaming
abstractions in Haskell (pipes, streaming, streamly being the others), and
none ever fully won. Conduit won the *ecosystem* battle — it underpins the
Yesod web framework, `http-conduit`, `xml-conduit`, `persistent`'s streaming
queries, and (historically) `amazonka` — so in practice you often use conduit
because a library you depend on already speaks it, not because you chose it.

The repo is a small mono-repo hosting `conduit`, `conduit-extra`,
`network-conduit-tls`, `cereal-conduit`, and `resourcet`. The API has been
essentially frozen since the 1.3 redesign in March 2018[^3]; releases since
then are bug fixes and new-GHC compatibility. Last push was June 2025 — at
~915 stars and 57 open issues, read this as feature-complete maintenance
mode, not abandonment: it remains in every Stackage LTS and Snoyman still
merges fixes, but no new direction is coming.

## Getting Started

```bash
cabal add conduit          # or add `conduit` to build-depends
# stack: add conduit to package.yaml dependencies
```

```haskell
import Conduit

main :: IO ()
main = do
  -- Pure pipeline: sum without materializing a list
  print $ runConduitPure $ yieldMany [1..10 :: Int] .| sumC

  -- Exception-safe, constant-memory file copy
  runConduitRes $ sourceFileBS "input.txt" .| sinkFile "output.txt"

  -- Transformations compose left-to-right, like a shell pipe
  print $ runConduitPure $
    yieldMany [1..10 :: Int] .| mapC (* 2) .| takeWhileC (< 18) .| sinkList
```

`runConduitPure` runs effect-free pipelines, `runConduit` runs them in a
monad, and `runConduitRes` adds `ResourceT` so file handles and sockets are
released promptly even on exceptions[^1].

## Architecture / How It Works

`ConduitT i o m r` has four type parameters: input consumed from upstream,
output sent downstream, base monad, and final result. A pipeline is a
component with `()` input and `Void` output, which `runConduit` collapses to
`m r`. Underneath sits a free-monad-style `Pipe` type whose constructors
represent the states a component can be in: produce a value, demand a value,
perform an effect, return leftovers, or finish. Composition (`.|`, a synonym
for `fuse`) interleaves two state machines; evaluation is **consumer-driven**
— nothing upstream runs until downstream demands a value, which is what makes
early termination (`takeWhileC`) stop upstream effects automatically[^1].

Key design points:

- **Leftovers.** A component can push unconsumed input back upstream
  (`leftover`), which parsers built on conduit (e.g. attoparsec adapters)
  rely on. Leftover semantics interact subtly with fusion: leftovers from a
  fused-away component are discarded, a classic source of confusion.
- **Chunked streams.** Byte and text streams flow as `ByteString`/`Text`
  chunks, not individual bytes. Combinators ending in `E` (`mapCE`,
  `filterCE`) operate element-wise within chunks; the un-suffixed versions
  operate on whole chunks.
- **Stream fusion.** The 1.2 series (2014) added a rewrite-rule fusion
  framework plus a codensity transform to avoid quadratic left-associated
  binds[^2]. Fusion only fires for combinator-style pipelines; hand-written
  `await`/`yield` loops fall back to the interpreted `Pipe` representation.
- **resourcet.** `ResourceT` maintains a mutable map of cleanup actions with
  ordered release. Conduit 1.3 removed pipeline-level finalizers entirely
  (simpler core, fewer guarantees) and delegated all prompt-cleanup
  responsibility to `ResourceT`[^3].
- **Applicative parallel consumption.** `ZipSink` feeds one stream to
  several sinks in a single pass; `ZipSource`/`ZipConduit` are the dual
  forms.

## Production Notes

**The chunk/element split is the #1 footgun.** `mapC` over a `ByteString`
stream maps over *chunks*, whose boundaries are an artifact of buffer sizes.
Any logic that assumes a chunk is a line or a record is a latent bug that
only appears with different chunking (network vs. file). Use `linesUnboundedC`
or the `E`-suffixed combinators, and test with pathological chunk sizes.

**Performance is adequate, not leading.** For IO-bound pipelines the
abstraction cost is noise. For tight CPU-bound loops, streamly and
hand-rolled recursion measurably beat conduit, and conduit's own performance
depends on `-O2` and on fusion rules actually firing — refactoring a
combinator chain into a monadic `do` block can silently deoptimize it.
Space leaks in accumulating components have recurred historically (e.g. the
`*>` leak fixed in 1.3.4.3[^2]).

**Prompt finalization requires `ResourceT` discipline since 1.3.** Resources
are released when `runResourceT`/`runConduitRes` exits, not necessarily the
moment a component finishes. For long-lived pipelines that open many
short-lived resources, allocate within narrower scopes or handle counts grow.

**Legacy syntax haunts tutorials.** Pre-1.3 code uses `$$`, `=$=`, `$=`,
`Source`, `Sink`, and `Conduit` type synonyms — all deprecated in 1.3.
Copy-pasting decade-old Stack Overflow answers produces deprecation warnings
or type errors; mentally translate everything to `ConduitT` and `.|`.

**Upgrade pain is behind you.** The 1.2→1.3 migration (2018) was the last
breaking change: dropped finalizers, `unliftio` instead of monad-control,
combinators folded in from `conduit-combinators`[^3]. Code written for 1.3
in 2018 still compiles against 1.3.6.1 — an unusually good stability record.

## When to Use / When Not

**Use when:**
- You're in the Yesod/`http-conduit`/`persistent` ecosystem — the streaming
  interface is already conduit, so fighting it costs more than adopting it.
- You process data larger than memory (logs, exports, uploads) and need
  guaranteed constant memory with exception-safe cleanup.
- You need one-pass multi-consumer folds (`ZipSink`) over a large stream.

**Avoid when:**
- Data fits in memory: plain lists, `Vector`, or strict folds are simpler
  and faster; conduit is "a bad list" when you don't need streaming[^1].
- You need maximum single-core throughput or built-in concurrency —
  composewell/streamly targets exactly that gap.
- You're not already in Haskell: this is an in-language abstraction, not a
  data-engineering tool.
- You value theoretical clarity over ecosystem reach — pipes has fewer
  concepts and laws-first design.

## Alternatives

- Gabriella439/pipes — use instead when you want a smaller, laws-driven core
  with bidirectionality; fewer batteries, less ecosystem pull.
- composewell/streamly — use instead when raw performance or built-in
  concurrent/reactive streams matter more than ecosystem compatibility.
- haskell-streaming/streaming — use instead when you want streams as an
  ordinary `FreeT`-style monad transformer with minimal new vocabulary.
- ekmett/machines — use instead for multi-input state-machine composition;
  more general, much smaller user base.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2011-12 | Initial release; replaces enumerator in the Yesod stack[^1]. |
| 1.0 | 2013 | API stabilization after rapid 0.x iteration. |
| 1.2 | 2014 | Stream fusion framework, codensity-based bind performance[^2]. |
| 1.3.0 | 2018-03 | Big cleanup: `ConduitT` + `.|` canonical, finalizers dropped, unliftio, combinators merged in[^3]. |
| 1.3.4.3 | 2023 | `*>` space-leak fix[^2]. |
| 1.3.6.1 | 2024 | Current: `mergeSource` fix, `-Wnoncanonical-monad-instances` forward compat[^2]. |

## References

[^1]: Conduit README/tutorial (updated for 1.3, March 2018). https://github.com/snoyberg/conduit#readme
[^2]: conduit ChangeLog. https://github.com/snoyberg/conduit/blob/master/conduit/ChangeLog.md
[^3]: conduit 1.3.0 changelog entry — finalizer removal, unliftio migration, deprecation of legacy operators. https://github.com/snoyberg/conduit/blob/master/conduit/ChangeLog.md#130

## Tags

haskell, streaming, data-pipeline, constant-memory, resource-management, monad-transformer, yesod-ecosystem, functional-programming, io
