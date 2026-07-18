# go-kit/kit

> A standard library for Go microservices: a three-layer transport/endpoint/service model that turns cross-cutting concerns into composable middleware.

[GitHub repo](https://github.com/go-kit/kit) ·
[Official website](https://gokit.io) ·
[License: MIT](https://github.com/go-kit/kit/blob/master/LICENSE)

## Overview

Go kit is a toolkit — not a framework — for building microservices (or, as its README puts it, "elegant monoliths") in Go. It was created by Peter Bourgon and first presented at GopherCon 2015[^1]. Its thesis was that Go had become the language of the server but lacked the coherent distributed-systems libraries that JVM shops relied on, so Go kit set out to provide logging, metrics, tracing, service discovery, rate limiting, and circuit breaking as a set of interoperable packages rather than a monolithic runtime.

The defining idea is the **endpoint**: every service method is reduced to a single function signature, `func(ctx, request) (response, error)`, and every cross-cutting concern is a middleware that wraps that function. This is a direct adaptation of Twitter Finagle's "your server as a function" model[^2]. The design is deliberately unopinionated about deployment, configuration, and orchestration — it expects to live inside a heterogeneous SOA where most services are not written with Go kit.

The tension that has defined Go kit's reception is **boilerplate versus explicitness**. The three-layer separation is clean and testable, but a single RPC method typically requires a request struct, a response struct, an endpoint adapter, and transport encode/decode functions — before any business logic is written. This verbosity spawned an ecosystem of code generators, and it is also the main reason many teams reach for lighter alternatives. The repository is stable but effectively in maintenance mode: it has never reached a v1.0, still sits at major version 0, and pushes have slowed markedly[^3].

## Getting Started

```bash
go get github.com/go-kit/kit
go get github.com/go-kit/log   # logging was split into its own module
```

A minimal service, endpoint, and HTTP transport:

```go
// service.go — business logic as a plain Go interface
type StringService interface {
	Uppercase(ctx context.Context, s string) (string, error)
}

type stringService struct{}

func (stringService) Uppercase(_ context.Context, s string) (string, error) {
	if s == "" {
		return "", errors.New("empty string")
	}
	return strings.ToUpper(s), nil
}

// endpoint.go — adapt the method into a generic endpoint
type upReq struct{ S string }
type upResp struct {
	V   string `json:"v"`
	Err string `json:"err,omitempty"`
}

func makeUppercase(svc StringService) endpoint.Endpoint {
	return func(ctx context.Context, request interface{}) (interface{}, error) {
		req := request.(upReq)
		v, err := svc.Uppercase(ctx, req.S)
		if err != nil {
			return upResp{v, err.Error()}, nil
		}
		return upResp{v, ""}, nil
	}
}

// main.go — wire an HTTP handler
func main() {
	svc := stringService{}
	h := httptransport.NewServer(
		makeUppercase(svc),
		func(_ context.Context, r *http.Request) (interface{}, error) {
			var req upReq
			return req, json.NewDecoder(r.Body).Decode(&req)
		},
		func(_ context.Context, w http.ResponseWriter, resp interface{}) error {
			return json.NewEncoder(w).Encode(resp)
		},
	)
	http.ListenAndServe(":8080", h)
}
```

## Architecture / How It Works

Go kit organizes a service into three concentric layers, often described as an onion:

1. **Transport layer** — HTTP, gRPC, Thrift, NATS, or AMQP. Its only job is to decode a wire request into a Go request value and encode the response back. Serialization and transport are pluggable by design; JSON-over-HTTP is a convention, not a mandate.
2. **Endpoint layer** — the pivot of the whole design. `endpoint.Endpoint` is `func(ctx context.Context, request interface{}) (response interface{}, error)`. Because every endpoint has the same signature, generic middleware can wrap any of them.
3. **Service layer** — your business logic, expressed as an ordinary Go interface with no Go kit types in its signatures. This keeps domain code independent of transport concerns and trivially unit-testable.

**Middleware** is the mechanism that makes the layers pay off. `endpoint.Middleware` is `func(Endpoint) Endpoint`, and there is an analogous `func(Service) Service` pattern for service-level decorators. Logging, instrumentation (Prometheus, statsd, and others via the `metrics` package), distributed tracing (OpenTracing/Zipkin), rate limiting (`golang.org/x/time/rate`), and circuit breaking (Sony's gobreaker or Netflix-style Hystrix) are all applied by wrapping, in a defined order, at composition time.

The `sd` package handles **service discovery and load balancing**, with subscribers for Consul, etcd, ZooKeeper, Eureka, and DNS SRV. It combines discovery with client-side load balancing and retry, so a caller resolves a set of instances and balances across them without an external proxy.

The most consequential internal decision is that requests and responses cross the endpoint boundary as `interface{}`. Go kit predates Go generics (added in Go 1.18) and never migrated to them, so type assertions and per-method struct plumbing are unavoidable. This is the direct source of the boilerplate that codegen tools try to erase.

## Production Notes

**Boilerplate is real and it compounds.** Every method multiplies into request/response structs plus endpoint and transport adapters. Teams above a handful of endpoints almost always adopt a generator — truss, microgen, mga, or protoc-gen-go-kit among them[^4] — and then inherit that generator's maintenance burden and abandonment risk (several listed generators are unmaintained). Budget for this before committing.

**The logging module moved.** The `log` and `metrics` logging pieces were extracted into a separate `github.com/go-kit/log` module. Older tutorials import `github.com/go-kit/kit/log`, which still exists but is a thin shim; mixing the two import paths in one build is a common source of confusion. Prefer `go-kit/log` directly.

**Still v0, but stable in practice.** The API has not made disruptive breaking changes in years, so v0 understates its actual stability. The flip side is that the project is largely dormant: releases are infrequent and the last significant push was mid-2024[^3]. Do not expect new transports, generics adoption, or active feature work.

**The stdlib has closed much of the gap.** Since Go 1.21–1.22 the standard library added `log/slog` for structured logging and pattern-based routing in `net/http`, removing two of Go kit's original justifications. For a small service, plain `net/http` plus `slog` now covers a lot of what Go kit was created to provide, with far less ceremony.

**Error semantics need care.** By convention, business errors are returned inside the response struct (an `Err` field) while transport errors use the function's error return. Getting this split wrong leads to HTTP 200 responses that carry application-level failures, or to failures that bypass your response encoders. Establish the convention early and enforce it in codegen or review.

**Context propagation is manual.** `context.Context` threads through every layer, which is idiomatic, but request-scoped values (auth, request IDs, deadlines) must be injected in transport decoders and read in endpoints deliberately — there is no ambient magic.

## When to Use / When Not

**Use when:**
- You are building many services that must share a uniform structure for logging, metrics, tracing, discovery, and resiliency, and you value that consistency over line count.
- You operate in a heterogeneous SOA and want transport-agnostic services (swap HTTP for gRPC without touching business logic).
- You want cross-cutting concerns expressed as explicit, testable middleware rather than framework magic.
- You are willing to invest in code generation to tame the boilerplate.

**Avoid when:**
- You are building one or a few small services — plain `net/http` + `slog` (Go 1.22+) is far less code.
- You want an actively evolving project; Go kit is in maintenance mode and has not adopted generics.
- Your team dislikes ceremony and type assertions, and would rather use generics-native or lighter routers.
- You only need RPC — use gRPC directly instead of layering Go kit over it.

## Alternatives

- micro/go-micro — a heavier, batteries-included microservice framework with its own registry/broker/runtime; use it when you want an opinionated platform, not a toolkit.
- grpc/grpc-go — use directly when RPC is your only real requirement and you don't need Go kit's middleware/discovery abstractions.
- gin-gonic/gin — use when you want a fast HTTP API with a conventional router and no endpoint/service layering.
- go-chi/chi — use when you want idiomatic `net/http` routing and middleware without adopting a new architecture.
- uber-go/fx — use when your primary need is dependency injection and lifecycle wiring rather than a service/transport model.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015-02 | Repository created; introduced at GopherCon 2015[^1]. |
| — | 2021 | `log` package extracted into the separate `go-kit/log` module. |
| v0.13.0 | 2023-01 | Latest tagged minor release; project stabilizes into maintenance mode. |
| — | 2024-07 | Last significant push as of this writing[^3]. |

## References

[^1]: Peter Bourgon, "Go kit: Go in the modern enterprise," GopherCon 2015. https://www.youtube.com/watch?v=1AjaZi4QuGo and https://peter.bourgon.org/go-kit/
[^2]: Marius Eriksen, "Your Server as a Function" (Twitter, PDF) — the Finagle model Go kit's endpoint design adapts. http://monkey.org/~marius/funsrv.pdf
[^3]: go-kit/kit repository metadata (created 2015-02-03, last push 2024-07-19, MIT, still major version 0). https://github.com/go-kit/kit
[^4]: Go kit README, "Code generators" section (truss, microgen, mga, protoc-gen-go-kit, and others). https://github.com/go-kit/kit#code-generators

## Tags

go, golang, microservices, toolkit, rpc, service-discovery, middleware, distributed-systems, grpc, backend, api
