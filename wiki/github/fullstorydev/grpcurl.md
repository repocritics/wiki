# fullstorydev/grpcurl

> Like cURL, but for gRPC — a command-line client that speaks protobuf-over-HTTP/2 so you can invoke and inspect gRPC services by hand.

[GitHub repo](https://github.com/fullstorydev/grpcurl) ·
[FullStory engineering blog](https://www.fullstory.com/resources/content/fullstory-engineering-blog/) ·
[License: MIT](https://github.com/fullstorydev/grpcurl/blob/master/LICENSE)

## Overview

`grpcurl` is a Go command-line tool for talking to gRPC servers the way `curl`
talks to HTTP servers. gRPC uses a binary protobuf wire format over HTTP/2, so a
plain `curl` cannot construct a valid request body or decode a response.
`grpcurl` closes that gap: you hand it JSON, it encodes to protobuf using the
service schema, invokes the method, and decodes the reply back to JSON. It also
acts as a schema browser — `list` and `describe` enumerate services, methods,
and message types.

The tool's defining dependency is the *descriptor source*. To translate JSON to
protobuf it must know the schema, and there are exactly three ways to give it
one: query the server's [reflection service](https://github.com/grpc/grpc/blob/master/doc/server-reflection.md),
point it at `.proto` source files, or feed it a compiled protoset (a binary
`FileDescriptorSet`). Everything about using `grpcurl` in practice reduces to
which of those three you have available, and reflection — the zero-config path —
is disabled by default on most production servers.

It began as an internal FullStory tool and was open-sourced in late 2017[^1]. The
heavy lifting — dynamic message construction, descriptor resolution, JSON⇄proto
transcoding — historically lived in the `jhump/protoreflect` library[^2] by the
same principal author, Joshua Humphries, who also wrote the companion web UI
`grpcui`. It is one of the most-installed pieces of gRPC tooling in existence,
packaged for Homebrew, Snap, Docker, and dozens of Linux distributions[^3].

## Getting Started

```bash
# macOS
brew install grpcurl
# from source (needs Go)
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
```

```bash
# List services on a reflection-enabled server (plaintext, no TLS)
grpcurl -plaintext localhost:8787 list

# Describe a method's request/response types
grpcurl -plaintext localhost:8787 describe my.pkg.Service.GetThing

# Invoke a unary method with a JSON body
grpcurl -plaintext -d '{"id": 1234, "tags": ["foo","bar"]}' \
    localhost:8787 my.pkg.Service/GetThing

# Server has no reflection? Supply the proto sources instead
grpcurl -import-path ./protos -proto my.proto \
    -d '{"id": 1234}' grpc.example.com:443 my.pkg.Service/GetThing
```

Note the argument ordering: all flags must come *before* the `host:port` and the
`Service/Method` symbol. Streaming request bodies are supplied by reading stdin
with `-d @`.

## Architecture / How It Works

`grpcurl` is a thin CLI (`cmd/grpcurl`) over a reusable Go library package
(`github.com/fullstorydev/grpcurl`) that other tools can embed to build dynamic
gRPC clients. The flow for any invocation:

1. **Resolve descriptors.** A descriptor source is built from reflection, proto
   files, or a protoset. With reflection, `grpcurl` opens a connection and calls
   the server's reflection RPC to pull the `FileDescriptorProto`s for the target
   symbol and its transitive dependencies.
2. **Parse the request.** JSON input is unmarshalled into a *dynamic* protobuf
   message built from the resolved descriptor — there is no generated Go struct
   for the target service, so everything is reflective.
3. **Invoke.** The dynamic message is sent over a standard `grpc-go` client
   connection; unary, server-streaming, client-streaming, and bidirectional
   streaming are all handled, the last of which can be driven interactively from
   a terminal.
4. **Format the response.** Reply messages are transcoded back to JSON (or text)
   using the protobuf JSON mapping.

The well-known `google.protobuf.*` types are baked into the binary as a
descriptor snapshot, so you never need to supply import paths for them — the same
convenience `protoc` provides. Reflection exists in two protocol versions
(`v1alpha` and the later `v1`); `grpcurl` speaks both and falls back between
them, which matters when talking to older servers that only registered the alpha
service.

The reflective, descriptor-driven design is the whole value proposition: no code
generation, no per-service binary, works against any server whose schema you can
obtain. It is also the source of every rough edge — everything depends on getting
a correct, complete descriptor set into the tool.

## Production Notes

- **Reflection is usually off.** Most hardened production servers do not register
  the reflection service (it exposes your full API surface). When `list` fails
  with "server does not support the reflection API," you need `.proto` files or a
  protoset — plan for that before an incident, not during one.
- **Protoset performance.** For scripted, repeated invocations, compile a
  protoset with `protoc --descriptor_set_out=... --include_imports` and pass
  `-protoset`. It skips proto parsing/compilation on every call. Forgetting
  `--include_imports` produces a protoset missing dependencies, which fails at
  runtime with confusing "not found" errors.
- **TLS is fiddly by design.** Plaintext servers require `-plaintext` explicitly
  (the default assumes TLS). Self-signed or custom-CA servers need `-cacert`;
  mutual TLS needs `-cert`/`-key`; and `-insecure` skips verification but still
  does TLS. Mixing these up is the most common first-run failure.
- **JSON⇄proto mapping surprises.** 64-bit integers serialize as JSON strings,
  enums accept either names or numbers, and unknown fields are rejected by
  default. Field names follow the protobuf JSON convention (lowerCamelCase)
  unless you pass `-use-proto-names`.
- **Not a load tester.** `grpcurl` invokes one call at a time. For benchmarking
  or concurrency use a purpose-built tool (see Alternatives). Docker users must
  remember `host.docker.internal` for loopback targets and `-i` for stdin bodies.

## When to Use / When Not

**Use when:**
- You need to manually invoke or smoke-test a gRPC method from a shell or script.
- You want to inspect an unfamiliar service's schema via reflection.
- You are building another CLI and want to embed dynamic gRPC invocation (use the
  library package directly).

**Avoid when:**
- You want an interactive REPL or a persistent, form-filling client experience —
  a dedicated interactive tool fits better.
- You need a graphical UI to explore and call services — reach for a web/GUI
  client instead.
- You are load-testing or benchmarking — grpcurl issues single calls, not
  concurrency.

## Alternatives

- fullstorydev/grpcui — same authors; browser UI over the same descriptor engine. Use when you want point-and-click exploration instead of a CLI.
- ktr0731/evans — interactive gRPC REPL with autocompletion. Use when you make many exploratory calls against one service in a session.
- bufbuild/buf — `buf curl` invokes gRPC/Connect/gRPC-Web from the Buf CLI. Use when you already manage schemas with Buf and want one toolchain.
- bojand/ghz — gRPC load-testing/benchmarking tool. Use when you need throughput and latency numbers, not a single call.
- postmanlabs/postman — has native gRPC support in the desktop app. Use when your team already lives in Postman collections.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017-11-20 | Repository open-sourced by FullStory[^1]. |
| 1.x | 2018–2020 | Stabilized CLI; protoset and proto-source descriptor modes matured. Featured in a GopherCon 2018 talk[^4]. |
| 1.8.x | 2021–2023 | Reflection `v1` support alongside `v1alpha`; ongoing `grpc-go` and protobuf runtime updates. |
| latest | 2026-07-08 | Active maintenance; most recent push to `master` at time of writing[^3]. |

*Exact minor-version release dates are omitted where not independently verified;
see the releases page for authoritative version history.*

## References

[^1]: fullstorydev/grpcurl repository, created 2017-11-20 (GitHub API metadata). https://github.com/fullstorydev/grpcurl
[^2]: `jhump/protoreflect` — dynamic protobuf reflection library underpinning grpcurl. https://github.com/jhump/protoreflect
[^3]: grpcurl packaging index and repository state (stars, forks, last push per GitHub API). https://repology.org/project/grpcurl/information
[^4]: "grpcurl" talk, GopherCon 2018. https://www.youtube.com/watch?v=dDr-8kbMnaw

## Tags

go, grpc, protobuf, cli, http2, rpc, developer-tools, networking, reflection, api-testing
