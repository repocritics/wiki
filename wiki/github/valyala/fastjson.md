# valyala/fastjson

> A parse-once, zero-allocation JSON reader and validator for Go that skips structs, reflection, and code generation.

[GitHub repo](https://github.com/valyala/fastjson) ·
[License: MIT](https://github.com/valyala/fastjson/blob/master/LICENSE)

## Overview

fastjson is a Go library for reading and validating JSON without binding it to
Go types. Instead of unmarshalling into a struct or `map[string]interface{}`, you
parse a byte slice into a lazily-navigable `*Value` tree and pull fields out by
path (`v.GetInt("foo", "0")`). It is one of several performance-oriented JSON
packages from Aliaksandr Valialkin, whose other projects (fasthttp, quicktemplate,
VictoriaMetrics) share the same allocation-avoidance philosophy[^1].

The defining tradeoff is stated plainly in the README: fastjson trades Go's
memory-safety ergonomics for speed[^2]. The Parser reuses its internal buffers
across calls, so a `*Value` (and everything reachable from it) is only valid until
the next `Parser.Parse` on the same parser. Hold a reference too long and you read
into a buffer that has already been overwritten — a class of bug the compiler
cannot catch. In exchange you get parsing that the project measures at up to ~15x
faster than `encoding/json`, with zero heap allocations on the hot path[^2].

fastjson is a reader, not a marshaller. It has no struct tags, no `Marshal`
equivalent for arbitrary Go values, and cannot read from an `io.Reader` (only from
strings/bytes already in memory, with a `Scanner` for concatenated JSON streams)[^2].
It is best understood as a specialized tool for high-throughput services that pick
a handful of fields out of large or numerous JSON payloads.

## Getting Started

```bash
go get -u github.com/valyala/fastjson
```

```go
package main

import (
	"log"

	"github.com/valyala/fastjson"
)

func main() {
	var p fastjson.Parser
	v, err := p.Parse(`{"str":"bar","int":123,"arr":[1,"foo",{}]}`)
	if err != nil {
		log.Fatal(err)
	}
	// GetStringBytes returns a []byte aliasing the input — copy if you keep it.
	log.Printf("str=%s int=%d arr.1=%s",
		v.GetStringBytes("str"), v.GetInt("int"), v.GetStringBytes("arr", "1"))
}
```

For one-off lookups there are package-level helpers (`fastjson.GetInt(data, "foo",
"0")`), but each call re-parses the whole input — use a `Parser` when reading more
than one field from the same document[^2].

## Architecture / How It Works

The core is a single-pass recursive-descent parser that builds a tree of `Value`
nodes, where each `Value` is effectively a tagged union over object, array, string,
number, bool, and null. Objects keep their members in insertion order (`Object.Visit`
iterates in original order), rather than in a hash map[^2].

The performance comes from buffer reuse, not from a fundamentally different parsing
algorithm. A `Parser` owns a pool of `Value` nodes and scratch buffers. On each
`Parse` it resets and refills those buffers rather than allocating fresh ones, which
is why steady-state parsing reports 0 allocs/op in the project's benchmarks[^2]. The
direct consequence is the library's central caveat: values returned by one `Parse`
alias memory that the next `Parse` on the same `Parser` will overwrite. The same
lifetime rule applies to `Arena`, the companion type for building JSON values.

String and byte accessors (`GetStringBytes`) return slices that point into the
original input or the parser's buffers — no copy is made — so callers that retain
them must copy explicitly. `Value.Get(...)` navigates to a subtree; `MarshalTo`
re-serializes any subtree; `Set` and `Del` mutate the in-memory tree, which is how
fastjson supports lightweight edit-and-reemit workflows without a full unmarshal/
marshal round trip[^2].

Because a `Parser` is a mutable, buffer-owning object, it is not safe for concurrent
use. The intended concurrency pattern is one `Parser` per goroutine, or a
`ParserPool` (`sync.Pool` under the hood) that hands parsers out and back[^2].

## Production Notes

- **Lifetime footgun is the #1 issue.** The most common bug reports reduce to holding
  a `*Value`, `GetStringBytes` slice, or `Object` past the next `Parse`/`Scanner.Next`
  on the same parser. If you need data to outlive the parse, copy it (`append([]byte(nil), b...)`
  or `string(b)`). Run tests under `-race`; the README explicitly recommends it[^2].
- **Not concurrency-safe.** A shared `Parser` across goroutines corrupts silently.
  Use `ParserPool` or per-goroutine parsers.
- **Memory is bounded by input size.** Parsing needs up to `sizeof(Value) * len(input)`
  bytes, so an attacker-supplied payload can force large allocations; cap request body
  size before parsing[^2]. fastjson aims never to panic on malformed input — it returns
  an error instead — but it does not defend against memory amplification for you.
- **No streaming from readers.** You must buffer the whole document into memory first;
  there is no incremental `io.Reader` decoder. `Scanner` only splits a string of
  back-to-back JSON values, it is not a bytestream parser.
- **Numbers.** Parsed lazily to `float64`/`int` on access; large integers and
  high-precision decimals inherit float64's limits unless you read the raw bytes yourself.
- **Maintenance cadence is slow, not dead.** The API has been stable for years and the
  repo still receives occasional commits; treat it as a finished, low-churn library
  rather than an actively evolving one. There are no tagged semantic-version releases —
  you pin by module pseudo-version/commit.

## When to Use / When Not

**Use when:**
- You read a few fields out of large or high-volume JSON (RTB bidding, JSON-RPC, log
  ingestion) and allocation/GC pressure from `encoding/json` is a measured bottleneck.
- The schema is dynamic, non-homogeneous, or unknown at compile time (e.g. arrays like
  `[123, "foo", [456], null]`) and defining structs is impractical.
- You want validation and field access in a single pass, unlike `gjson`/`jsonparser`.

**Avoid when:**
- You want ordinary, safe unmarshalling into typed structs — reach for `encoding/json`,
  `json-iterator/go`, `goccy/go-json`, or `bytedance/sonic`, which are drop-in and don't
  impose lifetime rules.
- You need to serialize arbitrary Go values (fastjson has no general marshaller).
- You parse from a stream/`io.Reader`, or you need to hand parsed values freely across
  goroutines and object lifetimes.

## Alternatives

- tidwall/gjson — simpler read-only path queries (`gjson.Get(json, "a.b.c")`); no
  validation and re-scans per query, but far fewer lifetime hazards.
- buger/jsonparser — similar zero-alloc field extraction with a callback API; also skips
  full validation.
- json-iterator/go — use when you want a near drop-in `encoding/json` replacement with
  struct unmarshalling and better speed.
- goccy/go-json — use when you want the fastest standard-compatible `Marshal`/`Unmarshal`
  with struct tags intact.
- bytedance/sonic — use on amd64 when you want SIMD/JIT-accelerated struct
  (de)serialization at scale and can accept its platform constraints.
- mailru/easyjson — use when you can code-generate marshallers from fixed structs ahead
  of time.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-05-28 | Repository created; parse-once `Value` API and validator[^1]. |
| — | ongoing | `Arena`/`Set`/`Del` mutation, `ParserPool`, and `Scanner` added over time; API long stable, no tagged releases[^1]. |

## References

[^1]: valyala/fastjson repository and commit history. https://github.com/valyala/fastjson
[^2]: fastjson README — features, known limitations, security notes, and benchmarks. https://github.com/valyala/fastjson/blob/master/README.md

## Tags

go, golang, json, json-parser, json-validation, zero-allocation, performance, parsing, library, no-reflection
