# ory/dockertest

> Boot ephemeral Docker containers from Go integration tests and tear them down when the test finishes.

[GitHub repo](https://github.com/ory/dockertest) ·
[Official website](https://www.ory.com) ·
[License: Apache-2.0](https://github.com/ory/dockertest/blob/v4/LICENSE)

## Overview

Dockertest is a Go library for integration testing against real services instead of mocks. You ask it for an image (`postgres:14`, `redis`, a locally built Dockerfile), it talks to the Docker daemon to start a container, hands back the randomly-assigned host port, and removes the container when the test is done[^1]. The pitch is straightforward: mocking a database/DBAL is brittle and drifts from reality, so test the real thing in a container that lives for seconds.

The library is maintained by Ory, the identity-infrastructure company, and has existed since 2015[^2] — it long predates and underpins Ory's own test suites (Kratos, Hydra, Keto). That heritage shows in the API: it is a thin, procedural wrapper over the Docker Engine API rather than a framework. There is a `Pool` (a connection to the Docker daemon plus bookkeeping) and a `Resource` (a running container). Everything else is convenience on top.

The defining tension is that dockertest is *simpler and lighter* than its main rival, testcontainers-go, and also *does less*. There is no module catalog of pre-wired services, no automatic orphan reaper daemon, no cross-language parity. For teams that want a couple of containers up during `go test` with minimal dependencies, that trade is the whole appeal. Teams that want batteries-included service modules tend to reach for testcontainers-go instead.

## Getting Started

```bash
go get github.com/ory/dockertest/v4
```

```go
package myapp_test

import (
	"testing"
	"time"

	dockertest "github.com/ory/dockertest/v4"
)

func TestPostgres(t *testing.T) {
	pool := dockertest.NewPoolT(t, "") // cleanup registered via t.Cleanup

	pg := pool.RunT(t, "postgres",
		dockertest.WithTag("14"),
		dockertest.WithEnv([]string{"POSTGRES_PASSWORD=secret", "POSTGRES_DB=testdb"}),
	)

	hostPort := pg.GetHostPort("5432/tcp") // e.g. 127.0.0.1:54320

	// Poll until the DB accepts connections; images are "started"
	// long before they are "ready".
	err := pool.Retry(t.Context(), 30*time.Second, func() error {
		// open a connection to hostPort and Ping(); return the error
		return nil
	})
	if err != nil {
		t.Fatalf("could not connect: %v", err)
	}
}
```

The `RunT`/`NewPoolT` helpers require a live Docker daemon reachable via the usual `DOCKER_HOST` resolution. `pool.Retry` is the single most important call: container start is not container readiness.

## Architecture / How It Works

Dockertest is a client of the Docker daemon; it runs no agent of its own. The two core types:

- **`Pool`** — holds the Docker API client and tracks the containers and networks it created. `NewPool(ctx, endpoint)` for library code (manual `Close`), `NewPoolT(t, endpoint)` for tests (auto-`Close` via `t.Cleanup`).
- **`Resource`** — one running container. Exposes `GetHostPort`, `GetPort`, `GetBoundIP`, `Exec`, `Logs`, `FollowLogs`, network attach/detach, and `Close`.

Ports are published to ephemeral host ports and read back after start, so parallel tests don't collide on fixed ports. Readiness is the caller's job via `pool.Retry` (fixed 1s interval, defaulting to `pool.MaxWait` = 60s) or the package-level `Retry` / `RetryWithBackoff` functions.

**The v4 rewrite is the architecturally significant event.** Two changes define it[^3]:

1. **Automatic, reference-counted container reuse.** Containers are keyed by `repository:tag` by default. Each `Run`/`RunT` increments a ref count; each `Close`/cleanup decrements it; the container is removed only when the last reference drops. Two tests requesting `postgres:14` share one container. `WithoutReuse()` opts out; `WithReuseID` sets a custom key. This is what makes v4 "significantly faster" than v3, which started a fresh container per call.
2. **A lightweight Docker client** replacing the full moby/`docker/docker` dependency. Historically dockertest pulled in the entire Docker Engine dependency tree — a frequent complaint because it bloated downstream `go.mod` graphs and caused version conflicts. v4 trims this substantially.

Building from a Dockerfile (`BuildAndRun` / `BuildAndRunT`), user-defined Docker networks for container-to-container tests, and typed sentinel errors (`ErrImagePullFailed`, `ErrContainerCreateFailed`, `ErrContainerStartFailed`, `ErrClientClosed`) round out the surface. Container internals are reachable through escape hatches — `WithContainerConfig` and `WithHostConfig` take modifier functions over the raw moby `container.Config` / `container.HostConfig`, so healthchecks, restart policies, memory/CPU limits, and bind mounts are all available without the library wrapping every field.

## Production Notes

**Docker must exist, and that is the whole game in CI.** On GitHub Actions `ubuntu-latest` the daemon is present and tests just run. On GitLab shared runners you need the `docker:dind` service and must point your app at host `docker` (not `localhost`) inside the `Retry` callback — a common first-run failure. Rootless Docker, Podman, Colima, and Docker Desktop on macOS/Windows all work but differ in socket location and networking; `DOCKER_HOST` and the bound IP returned by `GetBoundIP` are where surprises live.

**Leaked containers are the classic footgun.** If a test panics or the process is killed before cleanup runs, containers survive. The v3 mitigation was `resource.Expire(seconds)`, which sets a hard self-kill timer on the container so orphans die on their own — a pattern worth keeping even in v4 for CI robustness. Unlike testcontainers, dockertest ships no separate reaper daemon (Ryuk); leak protection is cooperative, not enforced by a sidecar.

**Reuse changes test isolation semantics.** v4's default reuse means two tests hitting `postgres:14` share state. That is faster but means one test can see another's data unless you namespace by database/schema, disable reuse for stateful cases with `WithoutReuse()`, or key with `WithReuseID`. Migrating a v3 suite to v4 without accounting for this can produce flaky, order-dependent failures.

**First run is slow; subsequent runs less so.** The initial image pull dominates wall time and is not something the library can hide. In CI, warm the Docker layer cache or pre-pull images in a setup step. `docker system prune -f` is the documented remedy when runners fill their disk with accumulated images and volumes.

**Version pinning matters.** The import path is `.../dockertest/v4`; the default branch on GitHub is `v4`. v3 remains widely deployed and is still importable at `.../dockertest/v3`, but new features land only on v4. See `UPGRADE.md` for the v3→v4 migration.

## When to Use / When Not

**Use when:**
- Your Go tests need a real Postgres/MySQL/Redis/Kafka/etc. rather than mocks.
- You want a small dependency footprint and a thin, explicit API over Docker.
- Docker is reliably available in local and CI environments.
- You already live in the Ory ecosystem or value a battle-tested, stable API.

**Avoid when:**
- Docker is unavailable or forbidden in your CI (locked-down runners, no privileged mode) — there is no fallback runtime.
- You want pre-built, maintained service modules and a cross-language story — testcontainers-go covers that far better.
- You need an enforced orphan reaper for guaranteed cleanup under crashes.
- Your tests are pure unit tests where a real service adds only latency.

## Alternatives

- testcontainers/testcontainers-go — the main competitor; richer module ecosystem, a Ryuk reaper for guaranteed cleanup, cross-language parity. Use it when you want batteries-included service modules over minimal dependencies.
- fsouza/go-dockerclient — lower-level Docker API client. Use it when you need direct daemon control and are willing to orchestrate test lifecycles yourself.
- docker/compose — declarative multi-service stacks. Use it when the fixture is a whole environment managed outside the test process, not per-test containers.
- moby/moby — the official Docker Go SDK. Use it when you are building tooling on the Engine API rather than testing against containers.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-03 | First release under Ory; Pool/Resource wrapper over the Docker API[^2]. |
| v3 | (long-stable) | The widely-deployed major line; fresh container per `Run`, full moby dependency, `Expire` for leak protection. |
| v4 | recent | Automatic reference-counted container reuse + lightweight Docker client; new default branch, `.../dockertest/v4` import path[^3]. |

## References

[^1]: Dockertest README — "Why should I use Dockertest?" and Quick Start. https://github.com/ory/dockertest
[^2]: Repository metadata: created 2015-03-19, maintained by Ory, Apache-2.0, Go. https://github.com/ory/dockertest
[^3]: Dockertest README — "Migration from v3" (automatic container reuse + lightweight docker client) and UPGRADE.md. https://github.com/ory/dockertest/blob/v4/UPGRADE.md

## Tags

go, integration-testing, docker, containers, testing, ci, ephemeral-environments, database-testing, test-fixtures, ory
