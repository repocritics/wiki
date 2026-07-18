# planetscale/vtprotobuf

> A `protoc` plug-in that generates reflection-free, allocation-light marshal/unmarshal Go code for Protocol Buffers APIv2.

[GitHub repo](https://github.com/planetscale/vtprotobuf) ·
[License: BSD-3-Clause](https://github.com/planetscale/vtprotobuf/blob/main/LICENSE)

## Overview

`vtprotobuf` is the `protoc-gen-go-vtproto` code generator extracted from the
Vitess project (the "vt" prefix) and maintained by PlanetScale[^1]. It solves one
problem: the standard `google.golang.org/protobuf` runtime encodes and decodes
messages with reflection, which is slow and allocation-heavy on hot paths.
`vtprotobuf` generates fully-unrolled `MarshalVT` / `UnmarshalVT` / `SizeVT`
methods per message type, so serialization becomes straight-line code with no
reflection and, in the marshal path, no intermediate allocations.

Crucially, it is not a fork of `protoc-gen-go` and does not replace it. It runs
*alongside* the upstream generator and emits extra `_vtproto.pb.go` files next to
the normal `.pb.go` output[^2]. The helpers are opt-in: your code (or an RPC
codec) must call `MarshalVT()` explicitly instead of `proto.Marshal()`. The design
lineage traces back to `gogo/protobuf`, whose unrolled-codegen approach
`vtprotobuf` reimplements for the APIv2 world after `gogo` went unmaintained[^3].

The defining tradeoff is scope discipline versus convenience. Because the helpers
are additive and opt-in, adoption is low-risk and reversible — but you get nothing
for free. Every message you want to accelerate must be reachable by the plug-in,
every RPC framework needs manual codec wiring, and mixing accelerated with
non-accelerated messages (common with third-party protos) forces you to write a
type-dispatching codec by hand.

## Getting Started

```bash
go install github.com/planetscale/vtprotobuf/cmd/protoc-gen-go-vtproto@latest
```

Run it as an extra plug-in in your existing `protoc` invocation. Your project must
already be on the APIv2 runtime (`google.golang.org/protobuf`); APIv1 generated
code is not supported.

```bash
protoc \
  --go_out=. --plugin protoc-gen-go="$(which protoc-gen-go)" \
  --go-vtproto_out=. --plugin protoc-gen-go-vtproto="$(which protoc-gen-go-vtproto)" \
  --go-vtproto_opt=features=marshal+unmarshal+size \
  proto/service.proto
```

```go
// Opt in explicitly on the hot path.
data, err := msg.MarshalVT()          // instead of proto.Marshal(msg)
if err != nil { /* ... */ }

out := &pb.MyMessage{}
if err := out.UnmarshalVT(data); err != nil { /* ... */ }
```

`buf` users can add `go-vtproto` as a plug-in in `buf.gen.yaml` and get the same
output from `buf generate`.

## Architecture / How It Works

The plug-in reads the `CodeGeneratorRequest` that `protoc` hands every plug-in and,
for each message, emits Go source that walks the message's fields directly. There
is no runtime type table: a `string` field at tag 3 becomes literal varint-tag +
length-prefix + copy code. Features are selected via `--go-vtproto_opt=features=`
and are independent generators:

- **`size`** — `SizeVT()`, an unrolled wire-size calculator. The marshal codegen
  calls it internally to pre-size the destination buffer.
- **`marshal`** — `MarshalVT()`, `MarshalToVT()`, and `MarshalToSizedBufferVT()`.
  The message is written back-to-front into a buffer sized by `SizeVT`, which is
  why there is a "sized buffer" variant. A `marshal_strict` variant orders fields
  by field number rather than the default reverse-declaration order.
- **`unmarshal`** — `UnmarshalVT()`. Note the merge semantic: if the receiver is
  not already zeroed, decoding merges into it rather than replacing (mirroring how
  `proto.Unmarshal` is defined as reset-then-merge). Call `proto.Reset` first, or
  use a fresh message, for replace semantics. An `unmarshal_unsafe`
  (`UnmarshalVTUnsafe`) variant aliases the wire buffer for `bytes`/`string` fields
  instead of copying — faster, but the buffer must outlive the message unmutated.
- **`pool`** — `ResetVT()`, `ReturnToVTPool()`, and `FromVTPool()` constructors
  backed by a per-type `sync.Pool`, for reusing message objects across requests.
- **`clone`** and **`equal`** — reflection-free `CloneVT()` and `EqualVT()`.

Generated code depends on a small runtime package in this repo, mainly for
optimized handling of protobuf well-known types (`Timestamp`, `Duration`, etc.)
embedded in your messages. A `unique` field option (Go 1.23+) interns strings via
`unique.Make` during unmarshal.

Because the helpers do not override the default marshaller, integration with RPC
frameworks is where the real wiring lives. For gRPC you register a
`codec/grpc.Codec{}` and blank-import the default proto codec so it gets replaced.
DRPC and Connect have their own hookup paths; Twirp has no codec hook at all and
the README recommends a `sed` search-and-replace over generated files.

## Production Notes

**You must handle mixed message types.** The moment your service serializes a
message that lacks `vtprotobuf` helpers — gRPC `Status`, `etcd` API messages, any
third-party proto — the naive `Codec` will fail or fall back incorrectly. The
practical pattern (and the one Vitess itself uses) is a custom codec that type-
asserts for the `MarshalVT`/`UnmarshalVT` interface and falls back to
`proto.Marshal` otherwise[^4]. Connect explicitly requires this because it
serializes `Status` internally.

**`unmarshal_unsafe` is a real footgun.** Aliasing the wire buffer means any code
that reuses or frees the receive buffer after decoding can silently corrupt
`string`/`bytes` fields. Only safe when you control the buffer's lifetime for as
long as the message is alive. `unmarshal_unsafe` also takes precedence over the
`unique` interning option.

**Pooling is manual and unforgiving.** `pool` must be enabled per message, either
via `option (vtproto.mempool)` in the `.proto` or an explicit
`--go-vtproto_opt=pool=<import>.<Message>` list. Using a message after
`ReturnToVTPool()` is undefined behavior; forgetting to return it is a
"performance leak," not a memory leak. This is the feature most likely to
introduce use-after-free-style bugs.

**Binary size and build coupling.** Unrolled codegen produces substantially more
Go source than the reflective path, inflating compile times and binary size on
large schemas. The `buildTag=<tag>` option lets you gate the generated files
behind a build tag so consumers can compile without them — recommended for
libraries, and it means downstream code must type-assert before calling `VT`
methods rather than assuming they exist. Reusing a non-zeroed message for decode
also merges rather than replaces (repeated fields append), which bites teams
pooling messages without `ResetVT`.

## When to Use / When Not

**Use when:**
- Protobuf (de)serialization is a measured hot path — high-QPS gRPC/DRPC services,
  streaming, or memory-pressure-sensitive workloads.
- You are already on APIv2 and control the `protoc`/`buf` pipeline.
- You want an opt-in, reversible optimization rather than a runtime replacement.

**Avoid when:**
- Serialization is not your bottleneck; the extra codegen, build weight, and codec
  wiring aren't worth it.
- You rely on protobuf reflection, dynamic messages, or `protojson` semantics —
  those still go through the standard runtime.
- Your message set is dominated by third-party protos you can't regenerate, so
  most traffic can't use the helpers anyway.
- You're still on the APIv1 runtime.

## Alternatives

- gogo/protobuf — the original unrolled-codegen approach `vtprotobuf` descends
  from; use it only for legacy APIv1 code, as it is effectively unmaintained.
- protocolbuffers/protobuf-go — the standard reflective runtime; use it when
  serialization is not a bottleneck and you want zero extra tooling.
- planetscale/vtprotobuf's sibling in other languages does not exist; for non-Go
  hot paths use flatbuffers/flatbuffers or capnproto/go-capnp instead when you can
  change the wire format.
- google/flatbuffers — use when you want zero-copy access and can abandon the
  protobuf wire format entirely.
- vmihailenco/msgpack — use when you don't need protobuf/gRPC and want a compact
  schema-less encoding.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-05-14 | Repo split out of Vitess as a standalone APIv2 plug-in[^1]. |
| — | 2021–2024 | Feature growth: `pool`, `clone`, `equal`, `marshal_strict`, `unmarshal_unsafe`, DRPC/Connect codecs. |
| — | 2024+ | `unique` string-interning field option (requires Go 1.23+)[^2]. |
| — | 2026-07 | Actively maintained; ~1.1k stars, steady commit activity, no tagged 1.0 — versioned via Go module pseudo-versions[^5]. |

## References

[^1]: PlanetScale, "vtprotobuf, the Vitess Protocol Buffers compiler" — repository README. https://github.com/planetscale/vtprotobuf
[^2]: vtprotobuf README, "Available features" and "Field Options" sections. https://github.com/planetscale/vtprotobuf#available-features
[^3]: gogo/protobuf — the unmaintained predecessor whose codegen approach vtprotobuf reimplements for APIv2. https://github.com/gogo/protobuf
[^4]: Vitess custom gRPC codec mixing vtprotobuf and non-vtprotobuf messages. https://github.com/vitessio/vitess/blob/main/go/vt/servenv/grpc_codec.go
[^5]: Repository metadata via GitHub API, retrieved 2026-07-18: 1,109 stars, 110 forks, BSD-3-Clause, last push 2026-07-02.

## Tags

go, protobuf, protoc-plugin, code-generation, grpc, serialization, performance, vitess, marshaling, rpc
