# connectrpc/connect-go

> Protobuf RPC built directly on net/http — one server handler speaks gRPC, gRPC-Web, and a plain HTTP/JSON protocol you can hit with curl.

[GitHub repo](https://github.com/connectrpc/connect-go) ·
[Official website](https://connectrpc.com) ·
[License: Apache-2.0](https://github.com/connectrpc/connect-go/blob/main/LICENSE)

## Overview

Connect is a library from Buf (the company behind the `buf` Protobuf tooling) for building RPC APIs from a Protocol Buffer schema[^1]. connect-go is the Go implementation: you write a `.proto` file, generate code with a plugin, implement an interface, and get a `net/http`-compatible handler plus a type-safe client. The defining feature is that a single handler simultaneously serves three wire protocols — gRPC, gRPC-Web, and Connect's own HTTP-based protocol — with content negotiation deciding which one each request uses[^2].

The design bet is different from grpc-go's. Where google.golang.org/grpc ships its own HTTP/2 stack, name resolver, and load-balancing APIs, connect-go is deliberately "just the standard library." A handler is an `http.Handler`, a client wraps an `http.Client`, and anything that already works with `net/http` — middleware, timeouts, TLS config, tracing wrappers, `httptest` — works unchanged[^2]. That is the whole thesis: fewer bespoke abstractions, at the cost of the batteries-included client-side load balancing and service discovery that grpc-go provides out of the box.

The second bet is the Connect protocol itself: a simple scheme over HTTP/1.1 or HTTP/2 where unary calls are ordinary POSTs with a JSON or binary body, so a browser can call the API without a proxy and a human can call it with curl[^3]. This makes Connect attractive for teams that want gRPC-style codegen and streaming but also need browser clients and easy debuggability, and unattractive for teams already fully invested in the gRPC ecosystem (xDS, channelz, existing interceptor libraries).

## Getting Started

```bash
# Add the runtime library (module path is connectrpc.com/connect)
go get connectrpc.com/connect

# Code generation is driven by buf + the connect-go plugin
go install github.com/bufbuild/buf/cmd/buf@latest
go install connectrpc.com/connect/cmd/protoc-gen-connect-go@latest
# plus protoc-gen-go for the message types
buf generate
```

```go
package main

import (
	"context"
	"net/http"

	"connectrpc.com/connect"
	greetv1 "example/gen/greet/v1"
	"example/gen/greet/v1/greetv1connect"
)

type GreetServer struct{}

func (s *GreetServer) Greet(
	ctx context.Context,
	req *connect.Request[greetv1.GreetRequest],
) (*connect.Response[greetv1.GreetResponse], error) {
	res := connect.NewResponse(&greetv1.GreetResponse{
		Greeting: "Hello, " + req.Msg.Name,
	})
	return res, nil
}

func main() {
	mux := http.NewServeMux()
	path, handler := greetv1connect.NewGreetServiceHandler(&GreetServer{})
	mux.Handle(path, handler)
	// h2c support for gRPC clients over cleartext lives in the http.Server config.
	http.ListenAndServe("localhost:8080", mux)
}
```

## Architecture / How It Works

Code generation produces one `*connect.go` file per service via `protoc-gen-connect-go`. It emits a handler constructor (`NewXServiceHandler`) that returns a route path and a plain `http.Handler`, plus a client constructor (`NewXServiceClient`) that takes an `http.Client` and a base URL[^2]. Message types themselves come from the standard `protoc-gen-go`; Connect only generates the RPC glue.

Requests and responses are wrapped in generics — `connect.Request[T]` / `connect.Response[T]` — which is how Connect exposes headers and trailers without polluting the Protobuf message. Streaming RPCs surface as `*connect.ServerStream`, `*connect.ClientStream`, or `*connect.BidiStream`. Because there is no bespoke transport, streaming semantics follow HTTP: full bidirectional streaming requires HTTP/2, while HTTP/1.1 supports unary and (half-duplex) server streaming only[^3].

Protocol selection happens per request from the `Content-Type` header: `application/grpc*` routes to the gRPC or gRPC-Web codec, everything else to the Connect protocol. The same handler answers all three, so migrating a gRPC service to also serve browsers is a codec concern, not a rewrite.

Cross-cutting behavior is handled by interceptors (`connect.WithInterceptors`), Connect's analog to gRPC interceptors — a single `UnaryFunc`/`StreamingFunc` chain covering logging, auth, retries, and metrics. Errors are modeled as `*connect.Error` carrying a `connect.Code` (the gRPC code set) and optional typed error details; Connect maps these onto HTTP status codes for the Connect protocol. Ancillary gRPC features that grpc-go bundles are split into separate modules: `connectrpc/grpcreflect-go` for server reflection and `connectrpc/grpchealth-go` for health checks, added only if you need them.

## Production Notes

- **No client-side load balancing or name resolution.** Because the client is an `http.Client`, you get whatever the standard library and your dialer provide. There is no built-in equivalent of grpc-go's round-robin over resolved endpoints, xDS, or subchannel management. Behind a headless service or L4 load balancer this is fine; for client-side LB across many pods you supply it yourself.
- **h2c is a common footgun.** gRPC clients expect HTTP/2. Serving HTTP/2 without TLS (h2c) requires explicit server configuration. Older code used `golang.org/x/net/http2/h2c`; recent Go (1.24+) exposes `http.Server.Protocols` / `http.Protocols` to enable unencrypted HTTP/2 without the wrapper. Forgetting this yields confusing failures where Connect and gRPC-Web work but native gRPC clients cannot connect.
- **Deadlines travel as headers.** Connect propagates context deadlines over the wire, but standard `http.Server`/`http.Client` timeouts still apply and can silently truncate long streams if misconfigured. The README explicitly warns that `http.DefaultClient` and `http.ListenAndServe` are not production settings — set read/write/idle timeouts and connection pools deliberately.
- **Import path migration.** The module was originally `github.com/bufbuild/connect-go`; the project was renamed Connect and moved to the `connectrpc` org, with the import path becoming `connectrpc.com/connect`[^4]. Old blog posts, Stack Overflow answers, and generated code reference the `bufbuild` path — an upgrade means rewriting imports and regenerating with the renamed plugin.
- **Interceptor ecosystem is smaller.** The grpc-go ecosystem has years of ready-made interceptors (auth, retry, ratelimit, otel). Connect has good OpenTelemetry support (`connectrpc/otelconnect-go`) but fewer third-party interceptors overall; expect to write more glue.
- **Version support is narrow by policy.** The module tracks only the two most recent Go releases (the ones still receiving security patches) and Protobuf APIv2. Staying current with Go is effectively required.

## When to Use / When Not

**Use when:**
- You need one backend to serve gRPC services and browser clients without running Envoy or a gRPC-Web proxy.
- You want Protobuf codegen and streaming but value curl-able, JSON-debuggable endpoints.
- You prefer to stay inside `net/http` and reuse existing HTTP middleware, tracing, and test tooling.
- You are also building a TypeScript frontend and want a matching client (connect-es).

**Avoid when:**
- You depend on the gRPC ecosystem's advanced features: xDS/service mesh integration, channelz, client-side load balancing, or a large library of existing gRPC interceptors.
- You have no browser or JSON requirement and are fully standardized on grpc-go — the extra protocol flexibility buys you little.
- You want the absolute simplest Protobuf-over-HTTP with no streaming and no gRPC compatibility; Twirp is smaller.

## Alternatives

- grpc/grpc-go — the canonical gRPC implementation; use when you need the full gRPC ecosystem (xDS, client-side LB, channelz, mature interceptors) and don't care about direct browser access.
- twitchtv/twirp — minimal Protobuf RPC over HTTP/1.1 with JSON support; use when you want simplicity and don't need streaming or gRPC wire compatibility.
- connectrpc/connect-es — the TypeScript/browser counterpart of Connect; use for the frontend client and Node servers alongside connect-go.
- grpc-ecosystem/grpc-gateway — REST/JSON transcoding generated alongside grpc-go; use when you already run grpc-go and want to bolt on a JSON API rather than switch libraries.
- connectrpc/vanguard-go — protocol transcoding middleware; use when you want a grpc-go service to also answer Connect/gRPC-Web/REST without changing its implementation.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.x | 2022 (H1) | Early access under `github.com/bufbuild/connect-go`, alongside the Connect launch[^1]. |
| v1.0.0 | 2022-09 | First stable release; semver stability promise for the 1.x line[^2]. |
| v1.10.0 | 2023 | Renamed to Connect; module path moved to `connectrpc.com/connect` under the `connectrpc` org[^4]. |
| 1.x | ongoing | Stable series; tracks the two latest Go releases and Protobuf APIv2. Last pushed 2026-07-15. |

## References

[^1]: Buf, "Connect: A better gRPC" — launch announcement. https://buf.build/blog/connect-a-better-grpc
[^2]: connect-go README and package docs. https://github.com/connectrpc/connect-go and https://pkg.go.dev/connectrpc.com/connect
[^3]: Connect protocol specification. https://connectrpc.com/docs/protocol
[^4]: Connect documentation, Go getting-started and module path (`connectrpc.com/connect`). https://connectrpc.com/docs/go/getting-started

## Tags

go, rpc, grpc, protobuf, grpc-web, net-http, api, codegen, streaming, buf, connect-protocol
