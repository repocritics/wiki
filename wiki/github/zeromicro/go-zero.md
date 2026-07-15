# zeromicro/go-zero

> A Go web + RPC framework where a code generator (`goctl`) and built-in resilience primitives are the product, not the router.

[GitHub repo](https://github.com/zeromicro/go-zero) ·
[Official website](https://go-zero.dev) ·
[License: MIT](https://github.com/zeromicro/go-zero/blob/master/LICENSE)

## Overview

go-zero is a Go microservices framework that pairs a small runtime (HTTP via the `rest` package, gRPC via `zrpc`) with a heavy code-generation CLI called `goctl`. It grew out of an internal system built in 2018 when a Chinese team migrated a Java + MongoDB monolith to Go microservices, and was open-sourced in August 2020[^1]. Its design premise is that most production incidents come from the same handful of failure modes — cascading timeouts, thundering herds, overload — so the framework ships adaptive circuit breaking, load shedding, and rate limiting on by default rather than as opt-in middleware.

The defining tension is generation vs. control. You describe a service in a small DSL (`.api` for HTTP, `.proto` for RPC) and `goctl` scaffolds handlers, routes, typed request/response structs, config, and a service-context wiring layer. This makes greenfield services fast and uniform across a large team, at the cost of living inside go-zero's directory conventions and regenerating (then merging) when the spec changes. Teams that value the generator love it; teams that want to hand-assemble their stack find the conventions opinionated.

go-zero is widely adopted in the Chinese Go ecosystem and is listed in the CNCF Landscape[^2]. With ~33k stars and active weekly releases through 2026, it is one of the most-used Go microservice frameworks, though its documentation and community discussion remain more mature in Chinese than in English.

## Getting Started

```shell
# runtime library
go get -u github.com/zeromicro/go-zero

# the code generator
go install github.com/zeromicro/go-zero/tools/goctl@latest   # or: brew install goctl
```

Describe a service in an `.api` file, then generate:

```go
// greet.api
type (
  Request  { Name string `path:"name,options=[you,me]"` } // auto-validated
  Response { Message string `json:"message"` }
)

service greet-api {
  @handler GreetHandler
  get /greet/from/:name (Request) returns (Response)
}
```

```shell
goctl api go -api greet.api -dir greet   # scaffold Go server
cd greet && go mod tidy && go run greet.go -f etc/greet-api.yaml
curl -i http://localhost:8888/greet/from/you
```

You then hand-write business logic in the generated `internal/logic/*.go` files and inject dependencies (MySQL, Redis, RPC clients) through `internal/svc/servicecontext.go`. The generator can also emit typed clients for Java, Kotlin, Dart, and TypeScript from the same `.api` file.

## Architecture / How It Works

**Two runtimes, one philosophy.** `rest` is a thin layer over `net/http` — it adds routing, parameter binding/validation from struct tags, and a middleware chain, but stays compatible with the standard `http.Handler`. `zrpc` wraps `google.golang.org/grpc`, adding go-zero's interceptors for the same resilience and observability concerns. Both read a single YAML config struct.

**goctl is the center of gravity.** Beyond API/RPC scaffolding, `goctl model mysql` and `goctl model mongo` generate data-access code from SQL schemas or collection definitions, including an optional cache layer with cache-aside logic, singleflight de-duplication, and cache-breakdown protection wired in. `goctl` templates are user-overridable, which is how teams enforce house style at scale.

**Resilience primitives** are the framework's real differentiator and run without configuration[^3]:
- **Adaptive circuit breaker** — a Google SRE-style client-side breaker that rejects requests probabilistically based on the recent success/failure ratio rather than a fixed error threshold.
- **Adaptive load shedding** — sheds inbound requests based on CPU usage and in-flight concurrency, protecting a process from overload before it falls over.
- **Rate limiting** — an in-process token-bucket limiter and a Redis-backed periodic limiter for distributed limits.
- **Timeout & concurrency control** — chained request timeouts and max-concurrency guards per service.

**Service discovery** defaults to etcd: `zrpc` servers register themselves and clients watch etcd for endpoint changes, doing client-side load balancing. Kubernetes-native discovery (via headless services / DNS) and direct-endpoint modes are also supported. Observability is built in: `logx` for structured logging, Prometheus metrics, and OpenTelemetry tracing propagated across `rest` and `zrpc` boundaries.

The coupling story: the runtime is genuinely modular (you can import `rest`, `zrpc`, or individual packages like `core/breaker` standalone), but the productive path assumes the generated project layout. Stepping off it means giving up most of `goctl`'s value.

## Production Notes

- **etcd is the default hard dependency.** The canonical `zrpc` discovery path expects an etcd cluster. Running go-zero RPC services without etcd (Kubernetes DNS, static endpoints) is supported but less documented, and much of the community tooling assumes etcd is present.
- **Regeneration merges are manual.** `goctl api go` regenerates handlers, types, and routes when the `.api` spec changes but does not touch your `logic/` files. In practice, spec changes require re-running the generator and reconciling by hand; there is no round-trip sync. Keep generated and hand-written code cleanly separated (the default layout does this) or merges get painful.
- **Cache correctness is your responsibility.** The generated model cache is cache-aside with sensible defaults, but invalidation on writes still follows go-zero's conventions; custom queries that bypass the generated methods will not participate in caching or its breakdown protection.
- **Load shedding can surprise you.** Adaptive shedding keys off CPU; on shared or CPU-throttled hosts (containers with tight CPU limits, noisy neighbors) it can begin rejecting requests earlier than expected. Tune or disable it deliberately for latency-insensitive batch workloads.
- **Documentation asymmetry.** The Chinese docs and community (Discord, issues) are consistently ahead of the English material. Non-Chinese-reading teams should expect to read source and generated code more than they would for a Western-origin framework.
- **Two-tier release stream.** The library (`v1.x.x`) and the `goctl` tool (`tools/goctl/v1.x.x`) are versioned and tagged separately in the same repo[^4]. Pin both; a matching goctl is what keeps generated code compatible with the runtime you import.
- **Validation is tag-driven.** Request validation comes from struct tags (`options`, `range`, `optional`) rather than a separate validator library. It covers common cases but is less expressive than dedicated validation ecosystems; complex cross-field rules still land in hand-written logic.

## When to Use / When Not

**Use when:**
- You're building many similar Go microservices with a team and want enforced uniformity and low boilerplate.
- You want production resilience (breaker, shedding, rate limit, tracing) without assembling and tuning it yourself.
- You already run etcd, or are comfortable with go-zero's spec-first, generate-then-fill workflow.
- You need typed clients in several languages generated from one API definition.

**Avoid when:**
- You prefer to hand-assemble your stack (chi/echo + your own gRPC setup) and dislike generated project layouts.
- You want a single HTTP router with no framework opinions — the standard library, chi, or gin are lighter.
- Your team can't read Chinese docs and the project is complex enough that English-only material will be a bottleneck.
- You need first-class, well-trodden non-etcd service discovery and don't run Kubernetes DNS-based routing.

## Alternatives

- cloudwego/kitex — ByteScale RPC framework; higher raw throughput and a strong Thrift/Protobuf story, less batteries-included web scaffolding.
- cloudwego/hertz — HTTP framework companion to Kitex; use when you want a fast router without go-zero's generator/resilience bundle.
- go-kratos/kratos — Bilibili's microservice framework; similar scope, config/DI-forward, use when you want conventions without goctl-style generation.
- grpc-ecosystem/go-grpc-middleware + gin-gonic/gin — assemble-your-own path; use when you want full control over each layer.
- micro/go-micro — pluggable microservice toolkit; use when you want swappable transports/registries over an opinionated generator.

## History

| Version | Date | Notes |
|---------|------|-------|
| open-sourced | 2020-08 | Extracted from an internal 2018 Go microservices system[^1]. |
| v1.0.0 | 2020-08-11 | First tagged release. |
| v1.1.0 | 2020-12-09 | Early stabilization of `rest` / `zrpc` and goctl. |
| v1.2.0 | 2021-09-13 | Ongoing generator and resilience refinement. |
| v1.3.0 | 2022-02-01 | — |
| v1.4.0 | 2022-08-07 | — |
| v1.5.0 | 2023-03-04 | — |
| v1.6.0 | 2023-10-28 | — |
| v1.7.0 | 2024-07-27 | — |
| v1.8.0 | 2025-01-28 | — |
| v1.9.0 | 2025-08-17 | — |
| v1.10.0 | 2026-02-15 | Current 1.10 line[^4]. |
| v1.10.2 | 2026-05-31 | Latest library release at time of writing[^4]. |

## References

[^1]: go-zero README, "Backgrounds of go-zero" — origin as a 2018 internal Java→Go microservices migration, open-sourced 2020. https://github.com/zeromicro/go-zero/blob/master/readme.md
[^2]: CNCF Cloud Native Landscape entry for go-zero. https://landscape.cncf.io/?selected=go-zero
[^3]: go-zero documentation — resilience (circuit breaker, load shedding, rate limiting). https://go-zero.dev
[^4]: go-zero releases — separate `vX.Y.Z` (library) and `tools/goctl/vX.Y.Z` (CLI) tag streams. https://github.com/zeromicro/go-zero/releases

## Tags

go, golang, microservices, rpc, grpc, web-framework, code-generation, cloud-native, resilience, service-discovery, rest-api
