# micro/go-micro

> One of Go's oldest microservice frameworks, repositioned in v6 as an "agent harness" where services double as AI-callable tools.

[GitHub repo](https://github.com/micro/go-micro) ·
[Official website](https://go-micro.dev) ·
[License: Apache-2.0](https://github.com/micro/go-micro/blob/master/LICENSE)

## Overview

Go Micro was created by Asim Aslam in 2015 as a pluggable framework for building
distributed systems in Go[^1]. For most of its life it was a microservice
toolkit: service registry, RPC client/server, pub/sub broker, and key-value
store, each exposed as a Go interface with swappable implementations. It arrived
alongside go-kit as one of the two reference answers to "how do I write
microservices in Go," leaning toward convention and batteries-included defaults
where go-kit leaned toward explicit composition.

As of v6 (2026) the project has been repositioned as an **agent harness**: the
same service abstractions now back AI agents, with every service endpoint
automatically exposed as an AI-callable tool over MCP, agents reachable over the
A2A protocol, and durable workflows (`flow`) for the deterministic parts[^2].
The pitch is that "an agent is a distributed system," so the microservice and
agent runtimes are the same code. This is a genuine pivot, not a side module —
the README, topics, and CLI all now lead with agents rather than services.

The defining tension is history versus reinvention. Go Micro carries a decade of
import-path and major-version churn (see Production Notes), and the v6 AI surface
is new code layered on a mature core. Teams evaluating it are really choosing
between two projects that share a name: the stable service framework and the
young agent harness.

## Getting Started

```bash
# Binary CLI (no Go toolchain required)
curl -fsSL https://go-micro.dev/install.sh | sh
# Or via Go
go install go-micro.dev/v6/cmd/micro@latest
```

A service is a struct whose methods become RPC endpoints. Doc comments and
`@example` tags become tool schemas for AI agents automatically:

```go
package main

import (
    "context"

    "go-micro.dev/v6"
)

type Request struct{ Name string `json:"name"` }
type Response struct{ Message string `json:"message"` }
type Say struct{}

// Hello greets a person by name.
// @example {"name": "Alice"}
func (h *Say) Hello(ctx context.Context, req *Request, rsp *Response) error {
    rsp.Message = "Hello " + req.Name
    return nil
}

func main() {
    s := micro.NewService("greeter")
    s.Handle(new(Say))
    s.Run()
}
```

`micro run` starts it with a dashboard, REST/gRPC endpoints, an MCP tool gateway,
and an agent playground on `:8080`.

## Architecture / How It Works

The core has always been a set of Go interfaces you compose and swap:

- **Registry** — service discovery; mDNS default (LAN-scoped), Consul/etcd/NATS for production.
- **Broker** — pub/sub (NATS, RabbitMQ, HTTP).
- **Transport / Client / Server** — gRPC transport, client-side load balancing, streaming.
- **Store** — key-value persistence (file/bbolt, Postgres, NATS KV); plus a `Codec`/`Selector` for wire format and node selection.

Historically these lived in a companion repo, `micro/go-plugins`, where most
non-default backends were implemented and where much version-skew pain
concentrated.

The v6 agent layer sits on top: `micro.NewAgent()` creates a service with an LLM
inside it and a proto-defined `Agent.Chat` RPC. It discovers its backing services
from the registry, scopes its tools to their endpoints, and keeps store-backed
conversation memory. Two gateways translate protocols without per-agent code: the
**MCP gateway** derives tools from service endpoints, and the **A2A gateway**
generates an Agent Card per registered agent and maps A2A tasks to `Agent.Chat`.
`micro.NewFlow()` provides checkpointed durable workflow steps, delegating to
agents for the unknown paths. `plan` and `delegate` are built-in agent tools;
`x402` adds opt-in per-call payments via a pluggable facilitator (no crypto in
the framework itself).

Agents, services, and flows share one runtime by design — the framework's thesis
and its risk: the AI abstractions inherit the maturity of the service core but
add a large, fast-moving surface on top.

## Production Notes

**Import-path and version churn is the historical footgun.** The module path has
moved repeatedly — `github.com/micro/go-micro` (v1), then `github.com/asim/go-micro`,
then the `go-micro.dev/vN` vanity path, now `go-micro.dev/v6` under the `micro`
org again[^3]. Each major bump is a distinct import path (Go modules semantics),
so upgrades are a codebase-wide find-and-replace, not a version bump, and old
tutorials frequently reference dead paths. The `asim/go-micro` URL redirects here.

**mDNS default discovery is for development.** The zero-dependency default does
not cross subnets and is not suitable for production clustering; a real registry
(Consul, etcd, or NATS) is expected before you deploy multi-host.

**Distinguish go-micro from the `micro` platform.** This repo (the library) is
Apache-2.0. The separate `micro/micro` runtime/CLI adopted a source-available
Polyform Shield license around 2020 that caused community friction[^4]; if
licensing matters, confirm which artifact you are pulling in.

**The v6 AI surface is young.** MCP/A2A/x402 support, agent memory compaction, and
durable flows are recent. Pin your major version and expect the agent APIs to move
faster than the service core. Relatedly, because backends historically lived in a
separate `go-plugins` repo, some implementations lag the current major version —
verify a backend is maintained against v6 before depending on it.

## When to Use / When Not

**Use when:**
- You want an opinionated, batteries-included Go framework where services,
  agents, and workflows share one runtime.
- You are building agent tooling in Go and want service endpoints exposed as MCP
  tools and A2A-reachable agents without wiring the protocols yourself.
- You prefer convention and pluggable defaults over assembling primitives.

**Avoid when:**
- You want a minimal, explicit RPC stack — use grpc-go or go-kit directly.
- You need long-term API stability with rare breaking changes; this project's
  history of path/version churn argues against it for conservative shops.
- Your agent/workflow needs are the core requirement and you want a mature,
  battle-tested engine specifically for that (Temporal for durable workflows).

## Alternatives

- go-kit/kit — explicit, composition-first microservice toolkit; more boilerplate, fewer surprises, no AI layer.
- grpc/grpc-go — use when you only want RPC and service definitions, not a full framework.
- temporalio/temporal — use when durable workflow execution is the core requirement and you want a dedicated, proven engine.
- encoredev/encore — use when you want an opinionated Go backend framework that also provisions infrastructure.
- mark3labs/mcp-go — use when you only need to build/serve MCP tools in Go without adopting a whole service framework.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1 | 2015 | Initial release as `github.com/micro/go-micro`; pluggable microservice framework[^1]. |
| v2 | 2020 | Go modules era; major refactor. Development later moved to `github.com/asim/go-micro`. |
| v3 | 2020 | Continued under `asim/go-micro`; `micro` platform licensing split occurred around this period[^4]. |
| v4 | ~2021 | `go-micro.dev/v4` vanity import path; long-stable service-framework line. |
| v6 | 2026 | Repositioned as an "agent harness": MCP/A2A gateways, agents, durable flows, x402 payments[^2]. Repo back under the `micro` org. |

## References

[^1]: Go Micro — original project by Asim Aslam. https://github.com/micro/go-micro
[^2]: Go Micro README — "an agent harness and service framework for Go." https://github.com/micro/go-micro/blob/master/README.md
[^3]: go.dev module reference, `go-micro.dev/v6`. https://pkg.go.dev/go-micro.dev/v6
[^4]: Micro (platform) licensing change discussion — Polyform Shield adoption, ~2020. https://github.com/micro/micro/blob/master/LICENSE

## Tags

go, golang, microservices, agent-harness, mcp, a2a, distributed-systems, rpc, ai-agents, service-framework
