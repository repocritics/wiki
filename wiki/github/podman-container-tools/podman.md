# podman-container-tools/podman

> A daemonless, rootless-capable engine for OCI containers and pods — a drop-in `docker` CLI without a central daemon.

[GitHub repo](https://github.com/containers/podman) ·
[Official website](https://podman.io) ·
[License: Apache-2.0](https://github.com/containers/podman/blob/main/LICENSE)

## Overview

Podman (the POD MANager) manages OCI and Docker container images, containers, volumes, and pods. It originated at Red Hat as the container-engine half of the `containers/` toolchain (Buildah, Skopeo, CRI-O, crun) and reached 1.0 in early 2019. Its defining architectural choice is the absence of a long-running daemon: `podman run` forks the container directly as a child of a small monitor process rather than sending a request to a root-owned `dockerd`. The repo is the home of both the `podman` CLI and `libpod`, the Go library that implements container lifecycle management.

The two things Podman is known for are **daemonless operation** and **rootless containers**. Rootless mode uses user namespaces so a normal user can run containers where in-container root maps to the unprivileged host user — a container can never hold privileges the launching user lacks[^1]. This is Podman's central security thesis and its central source of friction: rootless networking, storage, and port binding all behave differently from the root path, and many Docker tutorials assume behaviors that do not hold rootless.

The tradeoff to weigh: daemonless means no attack surface from a root daemon and no single point of failure, but it also means Podman has no built-in supervisor to restart containers on boot — that job is delegated to systemd (via Quadlet). Podman aims to be `docker`-compatible at the CLI and API level (`alias docker=podman` works for most workflows), but it is not Docker, and the divergences surface exactly where daemon semantics used to hide them.

## Getting Started

```bash
# Fedora/RHEL
sudo dnf install podman
# Debian/Ubuntu
sudo apt install podman
# macOS / Windows — Podman runs containers inside a managed VM
brew install podman        # then: podman machine init && podman machine start
```

```bash
# Rootless "hello world" — no daemon, no sudo
podman run --rm quay.io/podman/hello

# Run a detached service, map a port, name it
podman run -d --name web -p 8080:80 docker.io/library/nginx:alpine
podman ps
```

Group containers into a pod (shared network namespace, Kubernetes-style):

```bash
podman pod create --name app -p 8080:80
podman run -d --pod app docker.io/library/nginx:alpine
podman run -d --pod app docker.io/library/redis:7
podman kube generate app > app.yaml   # export as Kubernetes YAML
```

## Architecture / How It Works

Podman has no daemon. When you start a container, the flow is:

1. `podman` (or `libpod`) prepares the OCI runtime config and container storage.
2. It forks **conmon**[^2], a small C monitor, which execs the **OCI runtime** — **crun** (C, default) or **runc** (Go, alternative).
3. conmon double-forks the container so it is reparented to init and survives the `podman` process exiting. conmon holds the container's stdio, handles logging, detach, and exit-code capture.

The engine is deliberately a thin orchestrator over the shared `containers/` libraries:

- **Storage** — `containers/storage`, overlay driver (rootless uses native overlay on modern kernels, `fuse-overlayfs` on older ones).
- **Images / registries** — `containers/image`, which reads `registries.conf` (search registries, mirrors) and `policy.json` (signature trust) for pull/push, verification, and trust.
- **Networking** — **Netavark** (Rust) with **Aardvark-dns** for container DNS, which replaced CNI as the default in Podman 4.0[^3]. Rootless networking uses **pasta** (default since 5.0) or the older `slirp4netns`.
- **Builds** — Buildah's Go API is vendored in, so `podman build` needs no separate Buildah install.

Podman also exposes a **REST API** with two surfaces from one socket: a Docker-compatible API (so `docker-compose`, Testcontainers, and other Docker clients work by pointing at the Podman socket) and a richer libpod API for Podman-specific features like pods and checkpointing. On Linux the socket is a systemd-activated user or root unit; on macOS/Windows it is forwarded out of the `podman machine` VM.

**Pods** borrow the Kubernetes model: a pod is a group of containers sharing network (and optionally IPC/PID) namespaces via an infra container. `podman kube generate` and `podman kube play` round-trip a subset of Kubernetes YAML, making Podman a local approximation of a single-node kubelet.

## Production Notes

**Daemonless changes the restart story.** Without `dockerd`, `--restart` policies only take effect while Podman's own logic runs; surviving a reboot requires systemd. The current answer is **Quadlet**[^4] (stable since 4.4): you write declarative `.container`, `.pod`, `.network`, and `.volume` unit files under `/etc/containers/systemd/` and systemd generates real service units. `podman generate systemd` is the older path and is deprecated in favor of Quadlet. Rootless containers also need `loginctl enable-linger <user>` or they die when the user logs out.

**Rootless is where the footguns live.**
- Requires `/etc/subuid` and `/etc/subgid` entries for the user; a missing or too-small range breaks image extraction.
- Binding ports below 1024 needs `net.ipv4.ip_unprivileged_port_start` lowered, or rootlessport handling — otherwise `-p 80:80` fails as non-root.
- cgroups v2 is required for per-container CPU/memory limits rootless; on cgroups v1 hosts resource limits silently do less.
- Filesystem UID/GID inside volumes is shifted by the user-namespace mapping, which surprises anything that checks ownership of bind-mounted host files.

**`podman machine` adds a VM on Mac/Windows.** Everything runs in a Linux guest (applehv/krunkit on macOS, WSL2 on Windows). Consequences: bind-mount performance crosses a virtiofs/9p boundary, `localhost` port forwarding is proxied out of the VM, and container "host" networking is the VM, not your laptop. Treat the Mac/Windows experience as remote-over-VM, not native.

**Docker compatibility is high but not total.** Most `docker run` flags map directly, and the Docker API shim covers common tooling. Divergences to expect: default network and DNS behavior differ (Netavark vs Docker's bridge), healthcheck and event semantics are not identical, and Compose support depends on running the Docker-compatible socket — some Compose features map imperfectly. Podman also defaults to short-name pull resolution controlled by `registries.conf`, so an unqualified `nginx` may prompt or resolve differently than on Docker Hub-defaulted Docker.

**Upgrade pains.** The 4.0 network-stack switch from CNI to Netavark was the largest operator break — existing CNI networks needed migration, and custom CNI plugins stopped applying. The 5.0 move to pasta as the default rootless network changed rootless connectivity behavior for some setups.

## When to Use / When Not

**Use when:**
- You want containers without a root daemon — hardened, multi-tenant, or CI hosts where a privileged `dockerd` is unacceptable.
- You run rootless by policy and need in-container root mapped to an unprivileged user.
- You are on Fedora/RHEL/CentOS Stream, where Podman is the default, supported engine.
- You want systemd-native, declarative container units (Quadlet) instead of a separate orchestrator for single-node deployments.
- You want a local, Kubernetes-YAML-shaped workflow (`podman kube play`) before pushing to a real cluster.

**Avoid when:**
- Your tooling assumes a running Docker daemon and cannot be pointed at a socket (some Docker-Desktop-specific or daemon-event-driven workflows).
- You need mature, high-performance multi-node orchestration — that is Kubernetes' job, not Podman's.
- Your team is on macOS/Windows and expects native-speed bind mounts; the `podman machine` VM boundary will disappoint.
- You depend heavily on the Docker Compose ecosystem and cannot tolerate occasional feature gaps.

## Alternatives

- moby/moby — Docker's engine; the daemon-based incumbent with the largest ecosystem. Use when tooling hard-depends on `dockerd`.
- containerd/containerd — lower-level runtime beneath Docker and Kubernetes; use when you want a CRI/embeddable runtime, not an end-user CLI.
- containers/buildah — image building only; use when you need scripted, Dockerfile-free builds without a full container engine.
- cri-o/cri-o — Kubernetes-only container runtime (same `containers/` stack); use as the kubelet's CRI, not for local dev.
- lima-vm/lima — Linux VMs on macOS with containerd/nerdctl; use when you want a non-Podman rootless container VM on a Mac.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2019-01 | First stable release; daemonless `podman` + libpod. |
| 2.0 | 2020-06 | REST API introduced (Docker-compatible + libpod). |
| 3.0 | 2021-02 | Docker API compatibility maturation, stability focus. |
| 4.0 | 2022-02 | Netavark + Aardvark-dns replace CNI as default network stack[^3]. |
| 4.4 | 2023-02 | Quadlet — declarative systemd units for containers[^4]. |
| 5.0 | 2024-03 | pasta default for rootless networking; applehv default machine provider on macOS. |
| 6.x | 2026 | Current major line (Go module path `.../podman/v6`)[^5]. |

Podman ships a major or minor release four times a year (second week of February, May, August, and November), with more frequent patch releases; all releases are PGP-signed[^5].

## References

[^1]: Podman README, "Rootless" — user-namespace model, containers never exceed the launching user's privileges. https://github.com/containers/podman#rootless
[^2]: conmon — OCI container monitor used by Podman and CRI-O. https://github.com/containers/conmon
[^3]: Netavark, the Rust network stack that replaced CNI as Podman's default. https://github.com/containers/netavark
[^4]: Quadlet — running containers under systemd via generated units. https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html
[^5]: Podman README, release cadence and signing (four releases/year, PGP-signed); Go module major version v6. https://github.com/containers/podman#readme

## Tags

go, containers, oci, docker-alternative, rootless, daemonless, linux, kubernetes, pods, devops, containerization, red-hat
