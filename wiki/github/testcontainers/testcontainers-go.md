# testcontainers/testcontainers-go

> The Go port of Testcontainers — spin up real Docker dependencies inside a test and have them cleaned up automatically.

[GitHub repo](https://github.com/testcontainers/testcontainers-go) ·
[Official website](https://golang.testcontainers.org) ·
[License: MIT](https://github.com/testcontainers/testcontainers-go/blob/main/LICENSE)

## Overview

Testcontainers for Go is a library that lets an integration test declare the containers it needs — a Postgres, a Redis, a Kafka — start them programmatically against a real Docker daemon, wait until they are actually ready, and tear them down when the test finishes[^1]. It is the Go member of the Testcontainers family, which began on the JVM (Richard North, ~2015) and now has ports for Java, .NET, Node, Python, Go, Rust, and others. The Go port has been developed under the `testcontainers` org since 2018.

The pitch is fidelity over speed: instead of mocking a database or pointing tests at an in-memory fake, you test against the same engine you run in production. The cost is that every test now depends on a working container runtime, images have to be pulled and booted, and suites that used to run in milliseconds run in seconds. The defining tension is exactly this — real-dependency confidence versus test latency and the operational weight of requiring Docker everywhere the tests run, including CI.

The commercial context matters: AtomicJar, the company behind Testcontainers, was acquired by Docker, Inc. in late 2023. The library itself remains open source under MIT, but parts of the surrounding story (Testcontainers Cloud, Testcontainers Desktop) are commercial products from Docker.

## Getting Started

```bash
go get github.com/testcontainers/testcontainers-go
# module presets are separate Go modules:
go get github.com/testcontainers/testcontainers-go/modules/postgres
```

```go
package main

import (
	"context"
	"testing"

	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func TestWithRedis(t *testing.T) {
	ctx := context.Background()

	req := testcontainers.ContainerRequest{
		Image:        "redis:7",
		ExposedPorts: []string{"6379/tcp"},
		WaitingFor:   wait.ForListeningPort("6379/tcp"),
	}
	redis, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	if err != nil {
		t.Fatal(err)
	}
	testcontainers.CleanupContainer(t, redis) // stops + removes at test end

	endpoint, _ := redis.PortEndpoint(ctx, "6379/tcp", "")
	_ = endpoint // dial the mapped host:port, not 6379
}
```

## Architecture / How It Works

At the core is `ContainerRequest` (image, exposed ports, env, mounts, wait strategy) passed to `GenericContainer`. Under the hood the library talks to Docker through the official Docker Go SDK (the `moby/moby` client), so anything Docker-API-compatible — Docker Desktop, a remote `DOCKER_HOST`, Podman's Docker-compatible socket, Colima, Testcontainers Cloud — can be a backend[^2].

**Wait strategies** are the load-bearing abstraction. A container being "started" does not mean the service inside it is accepting connections, so `wait.ForListeningPort`, `wait.ForLog`, `wait.ForHTTP`, `wait.ForHealthCheck`, and `wait.ForSQL` block until a readiness condition is met. Skipping or mis-tuning the wait strategy is the most common source of flaky tests in this ecosystem.

**Ryuk (the resource reaper)** is the piece that makes automatic cleanup reliable. On first container creation the library starts a small sidecar container (`testcontainers/ryuk`) with the Docker socket mounted; the test process holds a connection to it and labels every resource it creates. If the test process dies — panic, `kill -9`, CI timeout — Ryuk notices the dropped connection and reaps the labelled containers, networks, and volumes so nothing is orphaned[^3]. It can be disabled with `TESTCONTAINERS_RYUK_DISABLED=true`, which matters in environments that forbid mounting the Docker socket.

**Modules** (`modules/postgres`, `modules/redis`, `modules/kafka`, `modules/mongodb`, `modules/localstack`, and many more) are pre-configured wrappers exposing typed helpers like `RunContainer` and `ConnectionString`. Each module is published as its own Go module with its own `go.mod` and version tag, so you pin them independently of the core library. Configuration also reads `~/.testcontainers.properties` and `TESTCONTAINERS_*` environment variables for daemon selection and Ryuk behavior.

## Production Notes

**Docker must exist wherever tests run.** This is the whole footgun surface. Local dev is usually fine; CI is where it bites. GitHub Actions Linux runners have Docker; macOS runners historically do not. Kubernetes-based CI often has no Docker socket at all, so you need Docker-in-Docker, a mounted host socket, or a remote `DOCKER_HOST` — and DinD interacts badly with Ryuk and with port mapping.

**Never hardcode the internal port.** Exposed ports are mapped to ephemeral host ports; you must resolve the mapped port via `MappedPort`/`PortEndpoint` (or a module's `ConnectionString`). Code that dials `localhost:5432` works on a laptop and breaks the moment two tests run in parallel or CI remaps the port.

**Host networking is not uniform.** On Docker Desktop (macOS/Windows) the daemon runs in a VM, so `localhost` inside a container is not the host; reaching the host requires `host.docker.internal` and `ExtraHosts`. Behaviour differs again under Podman/Colima/rootless Docker.

**Ryuk is a frequent CI casualty.** Restricted runners that block the Docker socket, or clusters where the sidecar cannot start, force `TESTCONTAINERS_RYUK_DISABLED=true` — at which point you own cleanup yourself. Always use `CleanupContainer`/`t.Cleanup` (or a `Terminate` deferred) rather than relying solely on Ryuk.

**Latency and images.** First run pulls images (cold, slow, network-dependent); cache them in CI. Startup is seconds per container, so heavy suites benefit from the experimental container **reuse** feature and from sharing one container across a package via `TestMain` instead of per-test containers. Disk usage from accumulated images/volumes is a real CI hygiene problem.

**Pre-1.0 versioning.** The core library has stayed on a `0.x` line, so minor bumps can carry breaking API changes; pin exact versions and read release notes before upgrading, and remember module versions move independently of the core.

## When to Use / When Not

**Use when:**
- Integration tests need a real Postgres/Kafka/Redis/S3, not a mock, and correctness against the real engine matters.
- You want deterministic setup/teardown in code instead of a hand-managed `docker-compose` sidecar.
- Your CI already has (or can have) a Docker daemon.

**Avoid when:**
- Your CI genuinely cannot run containers (locked-down Kubernetes, no DinD, no remote daemon).
- The suite is unit-level and mocks/fakes give enough confidence — the container tax isn't worth it.
- You need millisecond test loops; per-container startup will dominate.

## Alternatives

- ory/dockertest — older, lighter Go library that wraps the Docker API for tests; fewer batteries (no Ryuk-style reaper, no module presets), simpler mental model.
- orlangure/gnomock — preset-based container-for-tests library in the same niche; use when you want a smaller dependency surface.
- testcontainers/testcontainers-java — the original JVM implementation; use when your tests are in Java/Kotlin/Scala.
- Plain docker-compose in CI — use when containers are shared infra for the whole suite rather than per-test, and you don't need programmatic lifecycle control.
- Testcontainers Cloud — commercial remote-runtime backend; use when local/CI Docker is a bottleneck and you'll pay to offload it.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2018-07 | Repository created under the `testcontainers` org[^4]. |
| 0.x | ongoing | Long-running pre-1.0 line; API stabilized gradually, breaking changes across minors. |
| — | ~2023 | `modules/` presets introduced as independently versioned Go modules. |
| — | 2023-12 | AtomicJar (Testcontainers company) acquired by Docker, Inc.[^5] |

## References

[^1]: Testcontainers for Go documentation. https://golang.testcontainers.org
[^2]: Docker Engine API Go SDK (`moby/moby` client). https://pkg.go.dev/github.com/docker/docker/client
[^3]: Ryuk resource reaper. https://github.com/testcontainers/moby-ryuk
[^4]: testcontainers/testcontainers-go repository, created 2018-07-18. https://github.com/testcontainers/testcontainers-go
[^5]: Docker, Inc. acquires AtomicJar — 2023-12. https://www.docker.com/blog/docker-acquires-atomicjar-a-leader-in-integration-testing/

## Tags

go, golang, testing, integration-testing, docker, containers, test-fixtures, ryuk, ci, developer-tools
