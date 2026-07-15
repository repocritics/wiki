# grpc-ecosystem/grpc-gateway

> A protoc plugin that generates a reverse-proxy translating RESTful JSON HTTP calls into gRPC, from `google.api.http` annotations.

[GitHub repo](https://github.com/grpc-ecosystem/grpc-gateway) ·
[Official website](https://grpc-ecosystem.github.io/grpc-gateway/) ·
[License: BSD-3-Clause](https://github.com/grpc-ecosystem/grpc-gateway/blob/main/LICENSE)

## Overview

gRPC-Gateway is a code generator, not a library you call directly. It is a
plugin for the protocol-buffers compiler (`protoc`, or `buf`) that reads your
`.proto` service definitions and emits Go source for an HTTP/1.1 + JSON
reverse-proxy in front of a gRPC backend. Routing and payload shape come from
[`google.api.http`](https://github.com/googleapis/googleapis/blob/master/google/api/http.proto)
annotations — the same HTTP-mapping convention Google uses internally — so one
proto definition yields both a gRPC surface and a REST surface[^1].

The project exists to answer a specific tension: gRPC's tooling, streaming, and
wire efficiency are attractive, but a large fraction of clients (browsers,
webhooks, legacy integrations, curl-driven ops) still expect REST+JSON.
gRPC-Gateway lets a team commit to gRPC internally while keeping a JSON API for
consumers that cannot or will not speak protobuf over HTTP/2. It has been in the
grpc-ecosystem org for years and is one of the most-depended-on non-Google
pieces of the Go gRPC stack.

The defining tradeoff is that the REST surface is *derived*, not designed. You
get a JSON API for free, but it is shaped by protobuf semantics and the http
annotation grammar — not by REST idioms. Teams that want hand-crafted,
resource-oriented REST often find the generated API serviceable but not
idiomatic, and reach for the annotation escape hatches (custom marshalers,
header matchers, error handlers) more than they expected.

## Getting Started

Install the plugins (Go 1.24+ can track them via the `tool` directive in
`go.mod`; older projects use a `tools.go` blank-import file):

```sh
go install \
    github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway \
    github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2 \
    google.golang.org/protobuf/cmd/protoc-gen-go \
    google.golang.org/grpc/cmd/protoc-gen-go-grpc
```

Annotate a service, then wire the generated handler into an HTTP server:

```protobuf
service YourService {
  rpc Echo(StringMessage) returns (StringMessage) {
    option (google.api.http) = { post: "/v1/example/echo" body: "*" };
  }
}
```

```go
mux := runtime.NewServeMux()
opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
// dials the gRPC backend and proxies matching HTTP requests to it
err := gw.RegisterYourServiceHandlerFromEndpoint(ctx, mux, ":9090", opts)
if err != nil { return err }
http.ListenAndServe(":8081", mux)
```

## Architecture / How It Works

Generation produces a `*.pb.gw.go` file per proto containing `RegisterXHandler*`
functions. At runtime the work is done by the `runtime` package:
`runtime.ServeMux` is an `http.Handler` that pattern-matches incoming paths
against compiled templates from the http annotations, unmarshals the JSON body
and path/query parameters into the request protobuf, invokes the gRPC method,
and marshals the reply back to JSON. Default marshaling is `runtime.JSONPb`,
which follows protojson rules: fields are camelCase, enums serialize as their
string names, and 64-bit integers are encoded as strings.

There are two registration modes with materially different behavior:

- **`RegisterXHandlerFromEndpoint` / `RegisterXHandler`** — the gateway *dials*
  the gRPC server over the network. This is the documented default and the true
  reverse-proxy topology: the JSON request makes a second hop as a real gRPC
  call, so server interceptors, auth, and observability on the backend all run.
- **`RegisterXHandlerServer`** — registers the service implementation
  *in-process*, skipping the network hop. Faster, but it bypasses the gRPC
  client/dialer path, does not support client-streaming or bidi methods, and
  gRPC interceptors registered on the server do not fire the same way.

Cross-cutting behavior is customized through `ServeMux` options: incoming/outgoing
header matchers (which HTTP headers map to gRPC metadata and back), a custom
error handler (`runtime.WithErrorHandler`) to shape error JSON, and per-type
marshalers keyed by `Content-Type`. gRPC status codes map to HTTP status via
`runtime.HTTPStatusFromCode` unless overridden.

The `protoc-gen-openapiv2` plugin is a sibling generator that emits Swagger 2.0
(OpenAPI 2.0) documents from the same annotations. A newer
`protoc-gen-openapiv3` emits OpenAPI 3.1 but is explicitly **alpha**: the
maintainers warn its JSON output for oneofs, wrappers, and enums may change
between minor releases, and point to `protoc-gen-openapiv2` as the stable
choice[^2].

## Production Notes

- **The extra hop is real.** In the default `FromEndpoint` topology every REST
  request becomes a fresh gRPC dial+call to your backend. Budget for the added
  latency and connection pooling; run the gateway close to (or in the same pod
  as) the backend. The in-process mode removes the hop but with the streaming
  and interceptor caveats above.
- **Streaming is lopsided.** Server-streaming works over HTTP as a stream of
  newline-delimited JSON objects. Client-streaming and bidirectional streaming
  map awkwardly onto request/response HTTP and are a poor fit — do not expect a
  clean REST experience for them.
- **Error bodies need work.** The stock error response is a bare
  `{"code","message","details"}` shape from the default error handler. Most
  production deployments install `runtime.WithErrorHandler` to match their API's
  error contract; the defaults rarely satisfy an external API style guide.
- **Third-party proto deps bite with raw `protoc`.** Annotations pull in
  `google/api/annotations.proto`, `http.proto`, `field_behavior.proto`, etc.,
  which you must supply to the compiler manually. Using `buf` with the
  `buf.build/googleapis/googleapis` dependency removes almost all of this pain
  and is the recommended toolchain today.
- **Version skew is the classic footgun.** `protoc-gen-go`, `protoc-gen-go-grpc`,
  `grpc-go`, and the gateway must be kept on compatible versions. Mismatches
  surface as confusing generation or runtime errors; pin all four together.
- **JSON shape surprises consumers.** camelCase fields, string-encoded int64s,
  and string enums are protojson-correct but frequently trip up REST clients
  expecting snake_case or numeric enums. The `UseProtoNames` / `EmitUnpopulated`
  marshaler options exist for this, but they are opt-in.
- **Path-template collisions get renamed.** When two methods collapse to the
  same OpenAPI path (e.g. `{name=projects/*}` and `{name=organizations/*}` both
  becoming `/v1/{name}`), the generator disambiguates path params with a `_1`
  suffix, which can leak into generated clients.

## When to Use / When Not

**Use when:**
- You are committed to gRPC internally and need a REST+JSON surface for clients
  that cannot use gRPC (browsers, webhooks, third parties).
- Your proto is the source of truth and you want the HTTP API and OpenAPI spec
  generated from it rather than maintained by hand.
- You want to stay in the Go/protoc/buf toolchain without adding a separate
  proxy process.

**Avoid when:**
- You control both client and server and could adopt a single dual-protocol
  server (Connect) instead of running a translating proxy.
- You need hand-crafted, fully idiomatic REST semantics — the derived API will
  fight you.
- You already run Envoy and can enable its gRPC-JSON transcoder to do the same
  translation at the edge without code generation.
- Your API is streaming-heavy in the client-stream/bidi direction.

## Alternatives

- connectrpc/connect-go — one Go server speaks gRPC, gRPC-Web, and Connect's own
  HTTP/JSON protocol with no separate proxy or codegen-gateway; use when you own
  both ends and want browser-friendly RPC without a translation layer.
- envoyproxy/envoy — its gRPC-JSON transcoder filter performs the same
  annotation-driven translation at the proxy tier; use when Envoy is already in
  your data path and you prefer config over generated Go.
- grpc/grpc-web — exposes gRPC to browsers via a proxy but keeps the protobuf
  wire format; use when clients can speak gRPC-Web and you do not need REST/JSON.
- twitchtv/twirp — a simpler RPC framework serving JSON or protobuf over plain
  HTTP without gRPC/HTTP2; use when you want minimalism over the gRPC ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015 | Created (then under the `gengo`/personal namespace), later moved into the grpc-ecosystem org[^1]. |
| 1.x | 2015–2020 | Original series, built on the legacy `github.com/golang/protobuf` runtime. |
| 2.0 | 2020 | Module path moves to `/v2`; switches to the `google.golang.org/protobuf` runtime; plugin/runtime restructure[^3]. |
| 2.x | ongoing | `protoc-gen-openapiv2` (Swagger 2.0) stabilizes; `protoc-gen-openapiv3` (OpenAPI 3.1) added as alpha; SLSA3 signed releases[^2]. |

## References

[^1]: gRPC-Gateway README and documentation — "gRPC to JSON proxy generator following the gRPC HTTP spec." https://grpc-ecosystem.github.io/grpc-gateway/
[^2]: gRPC-Gateway docs, "OpenAPI 3.1 Output" and README alpha notice for `protoc-gen-openapiv3`. https://grpc-ecosystem.github.io/grpc-gateway/docs/mapping/openapi_v3/
[^3]: `google.api.http` HTTP-mapping annotation used to drive route generation. https://github.com/googleapis/googleapis/blob/master/google/api/http.proto

## Tags

go, grpc, rest-api, json, openapi, swagger, protobuf, code-generation, reverse-proxy, api-gateway
