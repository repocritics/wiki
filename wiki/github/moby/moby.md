# moby/moby

> The open-source engine behind Docker — a daemon, runtime glue, and toolkit for assembling container-based systems.

[GitHub repo](https://github.com/moby/moby) ·
[Official website](https://mobyproject.org/) ·
[License: Apache-2.0](https://github.com/moby/moby/blob/master/LICENSE)

## Overview

Moby is the upstream open-source project for Docker Engine. The repository was originally `docker/docker` — the codebase Docker Inc. open-sourced in 2013[^1] — and was renamed to `moby/moby` in 2017 when Docker split the community engine from its commercial products and reframed it as the "Moby Project," a modular toolkit for building container systems rather than a single end-user product[^2]. Docker continues to consume Moby as the upstream for Docker Engine; other projects are invited to reuse its components the same way.

The framing matters for anyone landing here expecting "Docker." Moby's README is explicit that it is aimed at engineers and integrators who want to hack on container internals, not at users seeking a supported product — for that, Docker Desktop or Mirantis Container Runtime are the commercial paths[^3]. Releases are best-effort, supported by maintainers and community.

The defining tension is that Moby is simultaneously a general-purpose "Lego set" and, in practice, the specific engine that ships as Docker. Much of the code assumes the Docker Engine assembly (the `dockerd` daemon, the Docker HTTP API, Swarm mode), so the modularity is real at the component boundary (containerd, runc, BuildKit are separate projects) but thinner at the daemon level. As of Docker v29 (November 2025) the project has restructured its Go modules and deprecated the long-standing `github.com/docker/docker` import path[^4].

## Getting Started

Moby is the engine source, not a product you install directly; most users install Docker Engine, which is built from it. On a Debian/Ubuntu host:

```bash
curl -fsSL https://get.docker.com | sh
docker run --rm hello-world
```

To use the maintained Go client against a running daemon (v29+ module paths):

```go
package main

import (
	"context"
	"fmt"

	"github.com/moby/moby/client"
)

func main() {
	cli, err := client.NewClientWithOpts(client.FromEnv, client.WithAPIVersionNegotiation())
	if err != nil {
		panic(err)
	}
	defer cli.Close()

	containers, err := cli.ContainerList(context.Background(), client.ContainerListOptions{})
	if err != nil {
		panic(err)
	}
	for _, c := range containers {
		fmt.Println(c.ID[:12], c.Image)
	}
}
```

The root module `github.com/moby/moby/v2` builds engine binaries and is explicitly not meant to be imported as a library — it has no API stability guarantee. The supported importable modules are `github.com/moby/moby/client` and `github.com/moby/moby/api`, versioned independently[^4].

## Architecture / How It Works

Docker Engine is not a monolith; it is a stack of layers that Moby coordinates:

1. **`dockerd`** — the daemon in this repo. It serves the Docker HTTP/REST API, manages images, networks (via libnetwork), volumes, and Swarm state, and delegates actual container execution downward.
2. **containerd** — a separate CNCF project ([containerd/containerd](https://github.com/containerd/containerd)) that `dockerd` talks to over gRPC. It owns the container lifecycle, image pull/unpack, and snapshotting. It was extracted from the Docker daemon and donated to the CNCF in 2017[^5].
3. **runc** — the low-level OCI runtime ([opencontainers/runc](https://github.com/opencontainers/runc)) that containerd invokes (via a `containerd-shim`) to actually create the container: setting up namespaces, cgroups, and the OCI spec. runc is the reference implementation of the Open Container Initiative runtime spec.

So a `docker run` traverses: Docker CLI → dockerd (this repo) → containerd → containerd-shim → runc → kernel (namespaces + cgroups). The shim keeps the container's parent process alive across daemon restarts, which is why `dockerd` can be restarted without killing running containers.

**Build** was historically the daemon's classic builder but is now delegated to **BuildKit** ([moby/buildkit](https://github.com/moby/buildkit)), which became the default builder in Docker 23.0 (2023)[^6]. BuildKit runs a DAG-based, concurrent, content-addressed build with better caching than the legacy sequential builder.

**Swarm mode** (orchestration) is built in via SwarmKit and was integrated into the engine in Docker 1.12 (2016). It coexists in the daemon but has seen little competitive investment against Kubernetes for years; treat it as maintained rather than evolving.

The co-evolution story is that Moby progressively pushed responsibilities out of the daemon (execution to containerd/runc, builds to BuildKit) while keeping the daemon as the API surface and assembly point. The pieces are genuinely reusable — Kubernetes talks to containerd directly via CRI and never touches `dockerd` — but the Docker Engine assembly remains the primary consumer of this repository.

## Production Notes

**The daemon is a single point of failure and a root-privileged one.** `dockerd` runs as root by default and exposes a socket that is effectively root-equivalent: anyone who can reach `/var/run/docker.sock` can mount the host filesystem and escalate. Never expose the daemon socket to untrusted containers or over TCP without mTLS. Rootless mode exists (stabilized in Docker 20.10) and mitigates much of this, at the cost of some networking and storage feature gaps.

**containerd is the real runtime — know the split.** Debugging execution problems often means dropping to `ctr` and inspecting containerd, not the Docker API. A hung `docker` command frequently reflects a containerd or shim issue, not the daemon proper. Kubernetes' 2022 removal of the `dockershim` (Docker Engine as a Kubernetes runtime) formalized what was already true: for orchestration, containerd is consumed directly and the Docker daemon is unnecessary overhead.

**Storage drivers matter.** `overlay2` is the default and correct choice on modern kernels. Legacy drivers (`devicemapper`, `aufs`, `btrfs`) carry footguns — `devicemapper` in `loop-lvm` mode in particular is a well-known production hazard. Inode and layer-count exhaustion on `/var/lib/docker` is a recurring operational surprise; `docker system df` and `docker system prune` are load-bearing.

**cgroups v2 and kernel coupling.** The engine's behavior is tied to host kernel features (cgroup version, overlay support, seccomp). Migrations from cgroups v1 to v2 have historically surfaced resource-limit and compatibility differences. The engine is only as stable as the kernel underneath it.

**The v29 module migration is a real break.** Starting with Docker v29 (November 2025), `github.com/docker/docker` is deprecated and frozen; consumers must migrate to `github.com/moby/moby/client` and `github.com/moby/moby/api`, and v29 ships many breaking Go SDK changes (renamed methods, option structs, moved types)[^4]. Code importing the old daemon internals as a library — always unsupported but common — will need rework.

## When to Use / When Not

**Use when:**
- You are building or customizing a container engine and want Docker's assembly as a starting point.
- You need the Docker Engine daemon and its API for local development or CI on a single host.
- You are contributing container-ecosystem tooling and want the upstream where Docker's engine changes land first.
- You want the batteries-included Docker experience (build, run, networks, volumes, Swarm) in one daemon.

**Avoid when:**
- You are running Kubernetes — use [containerd/containerd](https://github.com/containerd/containerd) or CRI-O directly; the Docker daemon adds no value there.
- You want daemonless or rootless-first containers — Podman's architecture fits better.
- You need commercial support or stability guarantees — this is a best-effort community upstream, not a product.
- You intend to import engine internals as a Go library — the root module explicitly offers no API stability.

## Alternatives

- [containerd/containerd](https://github.com/containerd/containerd) — the runtime Moby itself delegates to; use directly when you want the container lifecycle without the full Docker daemon (e.g. Kubernetes).
- [containers/podman](https://github.com/containers/podman) — daemonless, rootless-by-default, Docker-CLI-compatible; use when you want to avoid a long-running root daemon.
- [opencontainers/runc](https://github.com/opencontainers/runc) — the low-level OCI runtime; use when you need to spawn OCI containers with no higher-level engine.
- [moby/buildkit](https://github.com/moby/buildkit) — the build engine, usable standalone; use when you only need image builds, not a runtime.
- [cri-o/cri-o](https://github.com/cri-o/cri-o) — a lightweight CRI runtime for Kubernetes; use instead of the Docker daemon in a purely Kubernetes context.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013-03 | Docker open-sourced by dotCloud; repo `docker/docker`[^1]. |
| 1.0 | 2014-06 | First production-declared release. |
| 1.11 | 2016-04 | Runtime split: containerd + runc introduced under the daemon[^5]. |
| 1.12 | 2016-06 | Swarm mode integrated into the engine (SwarmKit). |
| 17.03 | 2017-03 | Date-based versioning; Docker CE/EE split. |
| — | 2017-04 | Moby Project announced; `docker/docker` → `moby/moby`[^2]. |
| 20.10 | 2020-12 | Rootless mode stable; long-lived LTS-style release. |
| 23.0 | 2023-02 | BuildKit becomes the default builder[^6]. |
| 29.0 | 2025-11 | Go module restructure; `github.com/docker/docker` deprecated[^4]. |

## References

[^1]: Solomon Hykes, "The future of Linux Containers" (PyCon 2013) / Docker open-source announcement, March 2013. https://www.docker.com/blog/
[^2]: Docker, "Introducing Moby Project" — 2017-04-18. https://www.docker.com/blog/introducing-the-moby-project/
[^3]: Moby README, "Audience" and "Relationship with Docker". https://github.com/moby/moby/blob/master/README.md
[^4]: Moby README, "Go modules" section, and Docker v29.0.0 release notes. https://github.com/moby/moby/releases/tag/docker-v29.0.0
[^5]: CNCF, "containerd project" (donated 2017). https://github.com/containerd/containerd
[^6]: Docker, "Docker Engine 23.0 release notes" — BuildKit as default builder. https://docs.docker.com/engine/release-notes/23.0/

## Tags

containers, docker, container-runtime, go, devops, oci, containerd, docker-engine, infrastructure, virtualization
