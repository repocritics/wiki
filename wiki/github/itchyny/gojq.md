# itchyny/gojq

> Pure Go reimplementation of jq — a drop-in CLI and, more importantly, an
> embeddable jq interpreter for Go programs.

[GitHub repo](https://github.com/itchyny/gojq) ·
[API docs (pkg.go.dev)](https://pkg.go.dev/github.com/itchyny/gojq) ·
License: MIT

## Overview

gojq is a from-scratch implementation of the jq query language in Go, started
by itchyny in April 2019. It ships as two things: a `gojq` binary that is a
near drop-in replacement for the `jq` command, and a Go package
(`github.com/itchyny/gojq`) that lets any Go program parse, compile, and run
jq queries against in-memory data[^1]. The library half is the real reason
the project matters: jq itself is a C program that is awkward to embed, so
gojq became the de facto jq engine for the Go ecosystem — wader/fq builds its
entire binary-format query tool on a gojq fork[^2], and Redpanda Connect
(formerly Benthos) uses it for its `jq` processor[^3].

The defining tradeoff is fidelity versus portability. gojq is deliberately
*not* bug-for-bug compatible with jq. It drops object key ordering (Go's
`map[string]any` is unordered, so output keys are always sorted and there is
no `keys_unsorted` or `-S` flag), swaps Oniguruma regexes for Go's RE2-style
engine (no backreferences, no look-around), and intentionally omits functions
like `$__loc__` and flags like `--ascii-output`[^1]. In exchange you get a
single static binary with no C library dependency, arbitrary-precision
integer arithmetic that jq lacks, and native YAML input/output. Scripts that
treat jq as a spec port cleanly; scripts that depend on key order or
Oniguruma regex features do not.

The project is actively maintained (last push July 2026) but is essentially
a single-maintainer effort, with a conservative cadence of a few patch
releases per year on the same 0.12.x line since December 2020[^4].

## Getting Started

```sh
brew install gojq          # macOS/Linux via Homebrew
go install github.com/itchyny/gojq/cmd/gojq@latest   # from source
```

```sh
$ echo '{"foo": 4722366482869645213696}' | gojq '.foo'
4722366482869645213696    # big integers keep full precision, unlike jq
```

As a library:

```go
query, err := gojq.Parse(".foo | ..")
if err != nil { log.Fatalln(err) }
iter := query.Run(map[string]any{"foo": []any{1, 2, 3}})
for {
    v, ok := iter.Next()
    if !ok { break }
    if err, ok := v.(error); ok { log.Fatalln(err) }
    fmt.Printf("%#v\n", v)
}
```

## Architecture / How It Works

gojq is a classic interpreter pipeline. A goyacc-generated parser (from
`parser.go.y`) plus a hand-written lexer turn query text into an AST; a
compiler lowers the AST into an internal instruction set; a stack-based
virtual machine executes it, emitting results through the `Iter` interface.
Parse and compile are separate public steps: `gojq.Parse` returns a `*Query`,
and `gojq.Compile` produces a reusable `*Code` so a hot path can compile a
query once and run it against many inputs[^5].

The data model is plain decoded JSON: inputs must be `any` values composed of
`map[string]any`, `[]any`, `string`, numbers, `bool`, and `nil` — exactly
what `encoding/json` produces. Custom structs and typed slices like `[]int`
are rejected; you must marshal to JSON and unmarshal back first[^5]. Numbers
use `int` and `math/big.Int` where jq uses C doubles, which is where the
arbitrary-precision behavior comes from — though only `+ - * %` (and exact
division) stay in integer land; `floor`, `round`, and every math function
convert to float64[^1].

Embedding is locked down by default: `env`/`$ENV` see no environment
variables, `input`/`inputs` are disabled, and module loading is off unless
opted in via compiler options (`WithEnvironLoader`, `WithInputIter`,
`WithModuleLoader`); custom Go functions are injected with `WithFunction` /
`WithIterFunction`[^5]. This makes gojq reasonable to expose to semi-trusted
query strings, which is precisely how tools like Benthos use it.

## Production Notes

- **Queries can run forever.** `repeat(0)`, `range(infinite)`, or a recursive
  `def f: f; f` will spin the iterator indefinitely. If query strings come
  from users, always use `RunWithContext` with a deadline — there is no other
  execution limit[^5].
- **Error handling is per-value, not per-run.** The iterator yields errors
  inline and may yield several; you must type-assert each result. `HaltError`
  with a nil value means "stop quietly" — the iterator does not terminate
  itself on halt, by design[^5].
- **The struct round-trip tax.** Inputs must be `encoding/json`-shaped, so
  querying domain types costs a marshal + unmarshal per input; in
  high-throughput pipelines this dominates. Benchmark before hot-path use.
- **Key order will differ from jq.** Output objects are always sorted by key.
  Diffing gojq output against jq output, or feeding it to systems that
  (wrongly) depend on insertion order, will surface this immediately[^1].
- **Regex dialect differences bite silently.** Go's engine rejects
  backreferences and look-around at compile time, but subtler flag and
  metacharacter differences from Oniguruma can change match results without
  erroring[^1].
- **Big-int output differs from jq.** Large integers print exactly instead of
  as lossy floats — usually an improvement, occasionally a golden-file
  breaker when migrating from jq.
- **Bus factor of one.** Single primary maintainer, stable but slow release
  rhythm (roughly quarterly patches)[^4]; worth knowing if you need a fix
  upstreamed quickly.

## When to Use / When Not

**Use when:**

- You need jq queries inside a Go program — this is the standard choice.
- You want a dependency-free static jq binary for containers or CI images.
- You need exact arithmetic on large integers, or YAML in/out with jq syntax.
- You are exposing user-supplied queries and want sandboxed-by-default
  env/module/input access.

**Avoid when:**

- Your scripts depend on object key order, `keys_unsorted`, or `-S`.
- You rely on Oniguruma regex features (backreferences, look-around).
- You need strict bug-for-bug jq compatibility for golden-file tests.
- You only need simple path extraction in Go — tidwall/gjson is leaner.

## Alternatives

- jqlang/jq — use the original when you need canonical behavior, key-order
  preservation, or Oniguruma regexes.
- wader/fq — use instead when querying binary formats (media, protobuf,
  archives); it embeds a gojq fork and extends the language.
- mikefarah/yq — use for YAML-first editing with in-place file updates and
  comment preservation, which gojq's YAML mode does not attempt.
- tidwall/gjson — use for fast, zero-parse JSON path lookups in Go when you
  do not need the jq language.
- jmespath/go-jmespath — use when the consumer side is standardized on
  JMESPath (AWS CLI ecosystem) rather than jq syntax.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.0.1 | 2019-04-14 | Initial release, two days after the first commit[^4]. |
| v0.11.0 | 2020-07-08 | Pre-0.12 maturation of CLI and library API[^4]. |
| v0.12.0 | 2020-12-24 | Start of the long-running 0.12 series[^4]. |
| v0.12.11 | 2022-12-24 | Steady patch cadence, ~quarterly releases on the 1st[^4]. |
| v0.12.19 | 2026-04-01 | Latest release as of mid-2026[^4]. |

## References

[^1]: gojq README, "Difference to jq". https://github.com/itchyny/gojq#difference-to-jq
[^2]: wader/fq — "jq implementation based on gojq". https://github.com/wader/fq
[^3]: Redpanda Connect (formerly Benthos) `jq` processor. https://github.com/redpanda-data/connect
[^4]: gojq releases. https://github.com/itchyny/gojq/releases
[^5]: gojq package documentation and README, "Usage as a library". https://pkg.go.dev/github.com/itchyny/gojq

## Tags

go, jq, json, yaml, cli, query-language, embeddable-library, interpreter, data-processing, arbitrary-precision
