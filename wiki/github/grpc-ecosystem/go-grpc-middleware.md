# grpc-ecosystem/go-grpc-middleware

> A collection of gRPC-Go interceptors — auth, logging, retries, recovery, rate limiting, validation — plus the plumbing to chain them.

[GitHub repo](https://github.com/grpc-ecosystem/go-grpc-middleware) ·
[License: Apache-2.0](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/main/LICENSE)

## Overview

gRPC-Go exposes a single extension point on both client and server: the *interceptor*, a function that wraps a call the way HTTP middleware wraps a handler. This repository is the community's standard bag of interceptors for the common cross-cutting concerns — authentication, structured logging, metrics, retries, timeouts, panic recovery, rate limiting, and message validation — along with the `selector` helper that turns any interceptor on or off per service/method. It sits under the `grpc-ecosystem` org alongside grpc-gateway, and is one of the most-depended-on Go modules in the gRPC world.

The project is on its second major line. **v1** dates from ~2015 and is effectively frozen; **v2** is the active branch (`main`) and is a deliberate rewrite that removed a lot of the surface area v1 accumulated[^1]. If you are starting today you want v2, and you should know before adopting that v2 is intentionally minimal: several things v1 shipped (logger adapters, OpenTracing, ctxtags, the chaining helper) were *deleted* rather than ported, on the reasoning that gRPC-Go or OpenTelemetry now own them.

The defining tension is scope. The maintainers are explicit that this is not a batteries-included framework — the README tells you it is "ok to copy some simpler interceptors" and that "this repo can't support all the edge cases you might have"[^2]. That honesty is the product: you get small, composable, low-dependency pieces, and you are expected to write the glue (logging adapters especially) yourself.

## Getting Started

The main module is v2, so the import path ends in `/v2`:

```bash
go get github.com/grpc-ecosystem/go-grpc-middleware/v2
```

```go
import (
	"google.golang.org/grpc"
	"github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/auth"
	"github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/logging"
	"github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/recovery"
	"github.com/grpc-ecosystem/go-grpc-middleware/v2/interceptors/selector"
)

srv := grpc.NewServer(
	grpc.ChainUnaryInterceptor(
		logging.UnaryServerInterceptor(myLogger),
		selector.UnaryServerInterceptor(
			auth.UnaryServerInterceptor(authFn),
			selector.MatchFunc(allButHealthZ), // skip auth on health checks
		),
		recovery.UnaryServerInterceptor(), // keep panic recovery last
	),
)
```

`myLogger` must satisfy `logging.Logger`, a one-method interface you implement over your logging library. The repo ships copy-paste example adapters for slog, zap, zerolog, logrus, logr, go-kit and the stdlib `log`, but they are examples to copy, not packages to import[^1].

## Architecture / How It Works

A gRPC interceptor is just a function. Server-side unary interceptors have the signature `func(ctx, req, info, handler) (resp, error)`; you do work, call `handler(ctx, req)`, and do more work on the way out. Streaming, and both client sides, have analogous signatures. Everything here is built on those four shapes plus `grpc.ChainUnaryInterceptor` / `grpc.ChainStreamInterceptor`, which gRPC-Go added natively — so v2 removed its own chaining helper[^1].

Two structural decisions shape the codebase:

- **Multi-module repository.** The core `interceptors/*` packages live in the v2 module and are kept "ultra slim" on dependencies. Anything that would pull in a heavy dependency lives under `providers/` as a *separate Go module with independent (v1-line) versioning* — most notably `providers/prometheus`, absorbed from the now-deprecated `go-grpc-prometheus`[^3]. The upside is that importing the retry interceptor doesn't drag Prometheus into your `go.mod`; the cost is that `providers/*` versions move independently of the main `v2.x.y` line and are easy to miss when upgrading.
- **The `Reporter` interface.** Generic observability interceptors are built on a small `Reporter` abstraction (`interceptors/reporter.go`), so metrics/logging-style middleware share one code path for post-call reporting rather than each re-implementing timing and status extraction.

The most important non-obvious piece is **`selector`**. In v1, most interceptors took a "decider" function (`WithDecider`, keyed on the full method name) to conditionally apply. v2 deleted all deciders and replaced them with one interceptor, `selector`, that wraps another interceptor and a `MatchFunc`[^1]. This is cleaner but it means the conditional-application pattern is now *external* to each interceptor — a real migration gotcha for anyone porting v1 config.

## Production Notes

- **Interceptor order is load-bearing.** Chains run in the order given. `recovery` must be last (innermost) or a panic in a later interceptor bypasses it — the README calls this out explicitly[^2]. Auth generally goes before logging so failed-auth requests are still logged; context-injecting interceptors (e.g. `logging.InjectFields`, trace-ID injection) must be chained *before* the code that reads those fields.
- **Logging is BYO adapter.** There is no importable logger package by design. You implement the `logging.Logger` interface once. Teams migrating from v1 are often surprised that `grpc_zap`/`grpc_logrus` no longer exist as packages — you copy the example and own it.
- **Provider module versioning.** `providers/prometheus` is versioned as `providers/prometheus/vX.Y.Z`, not in lockstep with `v2.x.y`. Pin it explicitly in `go.mod` and check its tags separately; a `go get -u` of the main module will not bump it.
- **Retries have a native competitor.** The `retry` interceptor predates and overlaps gRPC-Go's built-in retry policy (configured via service config / `MethodConfig`). For simple response-code retries the native mechanism may be enough and avoids a dependency; the interceptor is useful when you need per-call programmatic control. The README itself points at grpc-go's native retries[^2].
- **Validation split.** The old proto-option `validator` (codegen from `.proto`) coexists with a newer `protovalidate` interceptor built on Buf's protovalidate-go, which is where the ecosystem is heading; new services should generally prefer `protovalidate`.
- **Tracing lives elsewhere now.** v2 removed the OpenTracing interceptor. Use the official OpenTelemetry `otelgrpc` StatsHandler instead — it is a `StatsHandler`, not an interceptor, so it wires in via `grpc.StatsHandler(...)`, not the chain[^1].

## When to Use / When Not

**Use when:**
- You run gRPC-Go services and want vetted, low-dependency interceptors for auth, logging, recovery, retries, timeouts, rate limiting or validation.
- You want fine-grained per-method control via `selector` rather than one global policy.
- You value slim dependency graphs and are willing to write your own logging adapter.

**Avoid / reconsider when:**
- You only need metrics and tracing — the official OpenTelemetry `otelgrpc` StatsHandler covers those without this repo.
- You want a turnkey, batteries-included framework: the maintainers explicitly don't aim for that and suggest copying code for edge cases[^2].
- You're on gRPC-Go's native retry policy and don't need programmatic retry control — you may not need the `retry` interceptor at all.

## Alternatives

- grpc/grpc-go — the base library already provides native interceptor chaining, a retry policy, and an authz interceptor; reach here first before adding a dependency.
- open-telemetry/opentelemetry-go-contrib — use `otelgrpc` instead of this repo's (removed) tracing and, arguably, its metrics, when you're standardized on OpenTelemetry.
- bufbuild/protovalidate-go — the validation direction this repo's `protovalidate` interceptor wraps; use directly when you want validation outside the interceptor.
- grpc-ecosystem/go-grpc-prometheus — the original Prometheus interceptor, now deprecated and folded into `providers/prometheus` here; use the provider module, not the old repo.
- grpc-ecosystem/grpc-gateway — sibling project; different problem (REST↔gRPC transcoding), not a substitute, but the org you're already in.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.x | ~2015–2016 | Original interceptor collection; logger adapters, ctxtags, OpenTracing, deciders, own chaining helper. Now frozen. |
| v2 (dev) | 2020 onward | Rewrite: multi-module layout, `Reporter` interface, deciders replaced by `selector`, loggers/ctxtags/opentracing removed, new proto API[^1]. |
| v2.x.y | ongoing | Active main line; `providers/*` submodules versioned independently[^3]. |

Repository created 2016-05-14; still actively maintained (last push 2026-03).[^4]

## References

[^1]: "Changes compared to v1", project README — enumerates the v2 removals (loggers, opentracing, ctxtags, deciders, chaining) and the multi-module rationale. https://github.com/grpc-ecosystem/go-grpc-middleware#changes-compared-to-v1
[^2]: Project README, "Middleware" section — maintainers' explicit scope note ("ok to copy… can't support all edge cases") and recovery-last guidance. https://github.com/grpc-ecosystem/go-grpc-middleware
[^3]: README, "Structure of this repository" — `providers/` as separate Go modules; `providers/prometheus` absorbed from the deprecated go-grpc-prometheus. https://github.com/grpc-ecosystem/go-grpc-middleware#structure-of-this-repository
[^4]: GitHub repository metadata (stars, forks, license, last-push), retrieved 2026-07. https://github.com/grpc-ecosystem/go-grpc-middleware

## Tags

go, grpc, middleware, interceptor, authentication, logging, observability, retries, rate-limiting, validation, microservices, apache-2.0
