# grpc/grpc-go

> The pure-Go implementation of gRPC — HTTP/2 + Protocol Buffers RPC, with pluggable name resolution and load balancing built in.

[GitHub repo](https://github.com/grpc/grpc-go) ·
[Official website](https://grpc.io) ·
[License: Apache-2.0](https://github.com/grpc/grpc-go/blob/master/LICENSE)

## Overview

gRPC-Go is the Go implementation of gRPC, Google's cross-language RPC framework built on HTTP/2 with Protocol Buffers as the default serialization[^1]. Unlike the other major gRPC implementations (C++, Java, Python, and the many wrappers over the C core), the Go implementation is written entirely in Go — its own HTTP/2 transport, framing, and flow control, not a binding over `grpc/grpc`. The module path is `google.golang.org/grpc`, hosted here at `github.com/grpc/grpc-go`.

The library is the interface-definition-driven half of a two-step toolchain: you write a `.proto` service definition, run `protoc` with `protoc-gen-go` and `protoc-gen-go-grpc` to generate typed stubs, and this library provides the runtime those stubs bind to — the client dialer, the server, the streaming machinery, interceptors, and the resolver/balancer plumbing. The generated code is thin; almost all behavior lives in this repo.

Its defining tension is scope. What looks like "call a function on another server" carries an entire opinionated networking stack underneath: name resolution, client-side load balancing, keepalive, retry, and (via the `xds` package) full xDS control-plane integration for proxyless service mesh. That machinery is why gRPC-Go scales to Google-sized fleets, and also why a first RPC that returns `Unavailable` can have a dozen unrelated root causes. The framework has stayed on major version 1 for its entire modules life[^2] — there is no v2 module — which has kept the import path stable at the cost of accumulating deprecated-but-present API.

## Getting Started

```bash
go get google.golang.org/grpc
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
protoc --go_out=. --go-grpc_out=. helloworld.proto
```

```go
// client — grpc.NewClient is the current entry point; grpc.Dial is deprecated.
conn, err := grpc.NewClient(
    "dns:///localhost:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
if err != nil {
    log.Fatal(err)
}
defer conn.Close()

client := pb.NewGreeterClient(conn)
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "Tom"})
if err != nil {
    log.Fatalf("rpc failed: %v", err)   // inspect with status.FromError(err)
}
log.Println(resp.GetMessage())
```

## Architecture / How It Works

A `ClientConn` is not a single TCP connection — it is a managed abstraction over a set of connections to a set of resolved addresses, driven by two pluggable subsystems:

1. **Resolver** (`resolver.Builder`) — turns a target URI's scheme (`dns:///`, `passthrough:///`, `unix:///`, or a custom scheme) into a list of backend addresses, and watches for changes. `dns` is the default scheme under `grpc.NewClient`; `passthrough` (send the string straight to the dialer) was the historical default under the old `grpc.Dial`[^3].
2. **Balancer** (`balancer.Builder`) — decides which subconnection each RPC uses. `pick_first` is the default; `round_robin` and xDS-driven policies are configured via service config or dial options.

Below that sits the **HTTP/2 transport** (`internal/transport`), a hand-written client and server that manage streams, HPACK header compression, and per-stream/per-connection flow-control windows. Each RPC is one HTTP/2 stream. The four call shapes — unary, server-streaming, client-streaming, and bidirectional-streaming — are all the same stream primitive with different `SendMsg`/`RecvMsg` patterns.

Cross-cutting concerns are **interceptors**: `UnaryClientInterceptor`, `StreamServerInterceptor`, and their four siblings form the middleware layer for auth, logging, tracing, and metrics. Chaining is explicit and order-sensitive. Request-scoped key/value data rides in **metadata** (HTTP/2 headers and trailers), and every RPC carries a `context.Context` whose deadline propagates to the server as a header — cancel the context and the stream is cancelled on the wire.

The `xds` package layers Envoy's xDS APIs on top, so a gRPC-Go client can consume routing, weighted clusters, and mTLS config from a control plane (Istio, Traffic Director) with no sidecar proxy — "proxyless service mesh." This is the most complex optional surface in the library and pulls in a large dependency subtree.

## Production Notes

**`grpc.Dial` → `grpc.NewClient` migration.** `Dial`, `DialContext`, `WithBlock`, and `WithReturnConnectionError` are deprecated[^3]. `NewClient` does not eagerly connect and does not block — the first RPC triggers connection. Code that relied on `WithBlock` to fail fast at startup no longer does; health should be checked with an explicit RPC or the connectivity-state API. `NewClient` also changed the default resolver scheme to `dns`, which silently breaks callers that passed a bare `host:port` expecting `passthrough` semantics (e.g. a custom resolver or a load balancer address).

**Keepalive "too_many_pings."** Clients that send HTTP/2 keepalive pings more frequently than the server's `EnforcementPolicy.MinTime` allows get the connection torn down with a `GOAWAY` carrying `ENHANCE_YOUR_CALM` / `too_many_pings`. This is one of the most common self-inflicted outages; client `keepalive.ClientParameters` and server `keepalive.EnforcementPolicy` must be tuned together.

**`transport is closing` / `Unavailable`.** As the README itself documents, this error surfaces on the client but its cause is almost always server-side: mis-configured credentials, an intermediary proxy, server shutdown, or `MaxConnectionAge` recycling connections to force DNS re-resolution[^4]. Debugging requires logs on both ends (`GRPC_GO_LOG_SEVERITY_LEVEL=info`, `GRPC_GO_LOG_VERBOSITY_LEVEL=99`).

**Message size limits.** The default max receive message size is 4 MiB. Large payloads fail with `ResourceExhausted` until you raise `MaxRecvMsgSize` / `MaxCallRecvMsgSize` on both sides — but the better answer is usually streaming, not a bigger cap.

**Go version treadmill.** The project supports only the two most recent major Go releases[^5]. Because there is no v2 module, upgrades are frequent (roughly every six weeks) and low-drama, but staying on an EOL Go version eventually blocks security fixes.

**Generated-code split.** Service stubs come from `protoc-gen-go-grpc`, which was separated out from `protoc-gen-go` when the Protobuf APIv2 (`google.golang.org/protobuf`) landed. Mixing an old combined `protoc-gen-go` with a new runtime produces `undefined: grpc.SupportPackageIsVersion` errors; regenerate with the split plugins.

**Load balancing is client-side.** Unlike an L7 proxy model, gRPC-Go picks the backend in-process from the resolver's address set. A single long-lived `ClientConn` behind a plain L4 load balancer will pin to one backend — you need `round_robin` (or headless DNS / xDS) to actually spread load.

## When to Use / When Not

**Use when:**
- You control both client and server and want typed, versioned contracts with streaming.
- Internal service-to-service traffic where HTTP/2 multiplexing and low per-call overhead matter.
- You want client-side load balancing or service-mesh integration without a sidecar.
- Polyglot backends that must share one `.proto` contract across languages.

**Avoid when:**
- Browsers are first-class clients — native gRPC needs a proxy (grpc-web) or the Connect protocol instead.
- You want human-readable, curl-able, cache-friendly public APIs — REST/JSON or GraphQL fit better.
- The service is trivial and doesn't need streaming or codegen — the toolchain overhead outweighs the benefit.
- Your team can't budget for the operational learning curve (keepalive, resolvers, TLS, deadlines).

## Alternatives

- connectrpc/connect-go — gRPC-compatible services that also speak HTTP/1.1 and are browser-callable; use when you want gRPC semantics without the proxy and with simpler ergonomics.
- twitchtv/twirp — protobuf RPC over plain HTTP/1.1 with no streaming; use for internal services that value simplicity over gRPC's feature set.
- grpc-ecosystem/grpc-gateway — not a replacement but a companion: generates a REST/JSON reverse proxy in front of a gRPC server when you need both.
- go-kit/kit — transport-agnostic microservice toolkit; use when you want to abstract over gRPC, HTTP, and others behind one service definition.
- grpc/grpc — the C-core implementation backing C++, Python, Ruby, and others; use when your language isn't Go.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2014-12 | Initial gRPC-Go development, pre-1.0[^1]. |
| 1.0.0 | 2016-08 | gRPC 1.0 GA across languages; stable Go API. |
| APIv2 split | ~2020 | `protoc-gen-go-grpc` separated from `protoc-gen-go` alongside Protobuf APIv2[^6]. |
| xDS / proxyless mesh | ~2020–2021 | `xds` package matures for control-plane-driven routing and mTLS. |
| NewClient era | 2024 | `grpc.NewClient` introduced; `grpc.Dial`/`DialContext`/`WithBlock` deprecated[^3]. |

## References

[^1]: gRPC-Go repository and project site. https://grpc.io/docs/languages/go/
[^2]: Module path `google.golang.org/grpc`, major version 1 throughout its Go-modules history. https://pkg.go.dev/google.golang.org/grpc
[^3]: gRPC-Go, "Dial options and NewClient" — `Dial`/`WithBlock` deprecation and default-scheme change. https://github.com/grpc/grpc-go/blob/master/Documentation/anti-patterns.md
[^4]: gRPC-Go README FAQ, "transport is closing" and `MaxConnectionAgeGrace`. https://github.com/grpc/grpc-go/blob/master/README.md
[^5]: gRPC-Go supported Go versions policy. https://github.com/grpc/grpc-go#prerequisites
[^6]: `protoc-gen-go-grpc` plugin. https://pkg.go.dev/google.golang.org/grpc/cmd/protoc-gen-go-grpc

## Tags

go, golang, grpc, rpc, http2, protobuf, microservices, networking, load-balancing, service-mesh, xds, streaming
