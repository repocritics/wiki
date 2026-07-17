# twitchtv/twirp

> An RPC framework that generates Go clients and servers from protobuf service definitions and runs on plain `net/http` — gRPC's simplicity-first cousin, minus streaming and HTTP/2.

[GitHub repo](https://github.com/twitchtv/twirp) ·
[Official website](https://twitchtv.github.io/twirp/docs/intro.html) ·
[License: Apache-2.0](https://github.com/twitchtv/twirp/blob/main/LICENSE)

## Overview

Twirp is an RPC framework built at Twitch and open-sourced in January 2018[^1]. You
define services in protobuf, run `protoc` with the Twirp plugin, and get generated Go
server interfaces and clients. The deliberate design choice is to reuse the Go standard
library's `net/http` server rather than ship a custom transport the way gRPC does: Twirp
speaks ordinary HTTP/1.1, every call is a `POST`, and the wire body is either binary
protobuf or JSON depending on the `Content-Type` header[^2].

That decision is the whole product thesis. Twirp trades away gRPC's headline features —
bidirectional streaming, HTTP/2 multiplexing, deadline propagation, a large interceptor
ecosystem — in exchange for something you can debug with `curl`, put behind any HTTP load
balancer or proxy, and reason about without a specialized runtime. For request/response
service-to-service calls inside a Go backend, that is often the right trade. For anything
needing streaming, it is a hard non-fit, and no amount of configuration changes that.

Twirp is mature to the point of being quiet. The core spec has been stable for years and
the repository sees infrequent commits (last push mid-2024 as of this writing) — read
that as "finished and low-churn," not abandoned, but do not expect rapid feature work.
The Go repo here is canonical; implementations for other languages exist as third-party
projects of varying completeness[^3].

## Getting Started

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install github.com/twitchtv/twirp/protoc-gen-twirp@latest
```

```protobuf
// rpc/haberdasher/service.proto
syntax = "proto3";
package twitch.twirp.example.haberdasher;
option go_package = "example/rpc/haberdasher";

service Haberdasher {
  rpc MakeHat(Size) returns (Hat);
}
message Size { int32 inches = 1; }
message Hat  { int32 inches = 1; string color = 2; }
```

```bash
protoc --go_out=. --twirp_out=. rpc/haberdasher/service.proto
```

```go
// server: implement the generated interface, mount the generated handler.
package main

import (
	"context"
	"net/http"
	haberdasher "example/rpc/haberdasher"
)

type Server struct{}

func (s *Server) MakeHat(ctx context.Context, size *haberdasher.Size) (*haberdasher.Hat, error) {
	return &haberdasher.Hat{Inches: size.Inches, Color: "blue"}, nil
}

func main() {
	handler := haberdasher.NewHaberdasherServer(&Server{})
	http.ListenAndServe(":8080", handler) // handler is a plain http.Handler
}
```

## Architecture / How It Works

Code generation is split across two `protoc` plugins: `protoc-gen-go` produces the message
structs, and `protoc-gen-twirp` produces the service interface, the HTTP handler, and a
client. The generated server is an `http.Handler`, so it composes with any Go HTTP
middleware, mux, or server you already run.

Routing is fully mechanical. Every method maps to a single URL of the form
`/twirp/<package>.<Service>/<Method>`, always reached by `POST`[^2]. There is no path
templating, no query strings, no verb mapping — the protobuf definition is the routing
table. A request carries `Content-Type: application/protobuf` for binary or
`application/json` for JSON; the server decodes accordingly and encodes the response in
the same format. JSON support is what makes Twirp calls inspectable with `curl` and
readable in logs, at some CPU and payload-size cost versus protobuf.

Errors are a first-class, closed model. A handler returns a `twirp.Error` with one of a
fixed set of codes (`invalid_argument`, `not_found`, `internal`, `unauthenticated`,
etc.); each code maps to a defined HTTP status, and the error serializes as a JSON body
with `code`, `msg`, and optional `meta` regardless of the request format[^4]. This is
cleaner than ad-hoc HTTP status codes but less expressive than gRPC's rich status details.

Cross-cutting concerns are handled through `ServerHooks` and `ClientHooks` — callback
structs with fields like `RequestReceived`, `ResponseSent`, and `Error` — plus ordinary
`net/http` middleware wrapping the handler. There is no gRPC-style ordered interceptor
chain; observability and auth are assembled from hooks and HTTP middleware, and `context`
carries request-scoped values.

## Production Notes

- **No streaming, ever.** Twirp is unary request/response only. If a future requirement
  needs server-push, client-stream, or bidirectional, you are migrating frameworks, not
  toggling an option. Decide this up front.
- **POST-only breaks HTTP caching and REST tooling.** Because every call is a `POST` to an
  opaque path, GET-based CDN caching, browser prefetch, and REST-centric API gateways add
  little value. Twirp is for internal service RPC, not public REST APIs.
- **protoc toolchain version skew is the top footgun.** The move from the legacy
  `github.com/golang/protobuf` to `google.golang.org/protobuf` (protoc-gen-go v1 → v2)
  and mismatches between your installed `protoc`, plugin versions, and generated code are
  the most common breakage. Pin plugin versions (a `tools.go` + `go install` pattern) and
  regenerate in CI so contributors cannot drift.
- **Go semantic-import versioning matters on major upgrades.** Major versions use a
  versioned module import path; bumping across a major (e.g. v5 → v7) means updating import
  paths and regenerating, not just a `go get -u`. Check the release notes' upgrade guide.
- **Deadlines are not propagated on the wire.** Unlike gRPC, a client-side context timeout
  is not automatically transmitted as a server-enforced deadline; enforce timeouts at the
  HTTP layer and in server handlers explicitly.
- **Low repo activity.** The framework is stable and widely deployed, but issues and PRs
  can sit; treat it as a settled dependency and vet third-party language ports carefully,
  since their maturity varies a lot[^3].

## When to Use / When Not

**Use when:**
- You have a Go backend making internal service-to-service unary RPC calls.
- You want protobuf-defined contracts and generated clients without gRPC's operational surface.
- You value being able to `curl` an endpoint and read JSON responses while debugging.
- You want to run behind ordinary HTTP/1.1 infrastructure (L7 LBs, proxies, WAFs) with no HTTP/2 or ALPN requirements.

**Avoid when:**
- You need streaming of any kind, or HTTP/2 multiplexing and flow control.
- You're building a public REST API where GET/caching semantics and resource URLs matter.
- Your stack is polyglot and depends on the maintained, first-party multi-language support that gRPC or Connect provide.
- You want a rich interceptor ecosystem (tracing, retries, deadlines) out of the box.

## Alternatives

- grpc/grpc-go — use instead when you need streaming, HTTP/2, deadline propagation, or the broad first-party multi-language ecosystem.
- connectrpc/connect-go — use instead when you want Twirp-like ergonomics but with streaming, browser clients, and gRPC/gRPC-Web wire compatibility; effectively the modern successor in this niche.
- grpc-ecosystem/grpc-gateway — use instead when you already run gRPC and need a REST/JSON facade generated alongside it.
- goadesign/goa — use instead when you want a design-first DSL that emits both gRPC and REST transports from one description.
- twitchtv/twirp-ruby — use instead when you need a Twirp server/client outside Go and want a first-party Twitch-maintained port.

## History

| Version | Date | Notes |
|---------|------|-------|
| 5.0 | 2018-01 | Initial public release; announced by Twitch[^1]. |
| 7.x | 2020 | Routing/spec refinements; versioned Go module import path for the new major line. |
| 8.x | 2021+ | Current major line; protoc-gen-go v2 (`google.golang.org/protobuf`) and codegen updates. |

## References

[^1]: "Twirp: a sweet new RPC framework for Go" — Twitch blog, 2018-01-16. https://blog.twitch.tv/en/2018/01/16/twirp-a-sweet-new-rpc-framework-for-go-5f2febbf35f/
[^2]: Twirp protocol / routing documentation. https://twitchtv.github.io/twirp/docs/intro.html
[^3]: Third-party implementations table (README) — Crystal, Elixir, PHP, Python, Ruby, Rust, TypeScript, etc. https://github.com/twitchtv/twirp#implementations-in-other-languages
[^4]: Twirp errors documentation — error codes and HTTP status mapping. https://twitchtv.github.io/twirp/docs/errors.html

## Tags

go, rpc, protobuf, code-generation, microservices, grpc-alternative, net-http, service-framework, backend, protoc
