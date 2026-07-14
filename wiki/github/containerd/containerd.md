# containerd/containerd

> The container runtime that quietly sits under Docker, Kubernetes, and every major managed cloud — designed to be embedded, not driven by hand.

[GitHub repo](https://github.com/containerd/containerd) ·
[Official website](https://containerd.io) ·
[License: Apache-2.0](https://github.com/containerd/containerd/blob/main/LICENSE)

## Overview

containerd is a container runtime daemon that manages the full lifecycle of a container on a single host: image pull and storage, unpacking to a filesystem, process execution and supervision, and low-level storage/network attachment. It was extracted from the Docker Engine and donated to the CNCF in 2017[^1], reaching graduated status in 2019[^2]. Today it is the default runtime behind Docker Engine, and behind Kubernetes on every major managed platform (EKS, GKE, AKS) via its CRI plugin — especially after Kubernetes removed the built-in `dockershim` in 1.24 (2022)[^3].

The defining design choice, stated plainly in the README, is that containerd is meant to be *embedded into a larger system, rather than used directly by developers or end-users*[^4]. It deliberately does not build images, does not manage networks at a high level (it delegates to CNI), and does not ship a friendly CLI. The bundled `ctr` tool is a debugging aid, not a user interface. This narrow scope is what let it become universal plumbing — the same daemon underneath Docker and Kubernetes — but it also means containerd alone is not a complete container experience.

The tension for operators: containerd is extremely stable and boring by intent, yet almost nobody runs it as a first-class product. It is a dependency that appears in incident postmortems more often than in deliberate architecture decisions, which means teams frequently debug a runtime they never chose to install.

## Getting Started

Most users get containerd through a distro package or a Kubernetes installer, not by building it. For direct experimentation:

```bash
# Debian/Ubuntu — the distro package
sudo apt-get install -y containerd
sudo systemctl enable --now containerd

# Pull and run with the debug CLI (ctr), inside a namespace
sudo ctr images pull docker.io/library/redis:alpine
sudo ctr run --rm -t docker.io/library/redis:alpine redis
```

For a Docker-like experience on top of containerd, use `nerdctl` (a sibling containerd project) rather than `ctr`:

```bash
nerdctl run -d -p 8080:80 nginx:alpine
```

Requires `runc` on Linux and a reasonably modern kernel (4.x is a practical floor; the default overlayfs snapshotter depends on 4.x-era features)[^4].

## Architecture / How It Works

containerd is a long-running daemon exposing a gRPC API over a Unix socket. Internally almost everything is a plugin — snapshotters, runtimes, content store, and the services themselves are registered plugins, which is why `containerd config` output is dominated by plugin stanzas. The main subsystems:

- **Content store** — content-addressable blob storage for image layers and manifests, keyed by digest.
- **Snapshotters** — the copy-on-write filesystem layer. `overlayfs` is the default; `btrfs`, `zfs`, `devmapper`, `native`, and remote/lazy-pull snapshotters like stargz are pluggable alternatives. Choice of snapshotter is one of the highest-impact tuning decisions.
- **Metadata store** — a `bbolt` (embedded key-value) database holding namespaced references to images, containers, and snapshots.
- **Runtime / shim** — containerd does not run containers directly. It launches a **shim** process (`containerd-shim-runc-v2`) per container or pod, which in turn calls an OCI runtime (`runc` by default) to create the process[^5].

The shim architecture is the most important operational property. Because each container's shim is a separate process that reparents the container, the containerd daemon can be restarted or upgraded without killing running containers — the shims keep supervising and reconnect. This decoupling is why in-place containerd upgrades are safe in a way that monolithic runtimes are not.

- **CRI plugin** — a native plugin (built in and enabled by default since 1.1) that implements the Kubernetes Container Runtime Interface, so kubelet talks to containerd directly. It is the busiest code path in practice[^4].
- **Namespaces** — a containerd-level multi-tenancy boundary (unrelated to Linux namespaces). Docker uses `moby`, Kubernetes uses `k8s.io`; objects in one are invisible to the other, which routinely confuses operators debugging with `ctr` in the wrong namespace.

## Production Notes

- **The `ctr` / namespace trap.** `ctr images ls` on a Kubernetes node shows nothing useful unless you pass `-n k8s.io`. Use `crictl` (CRI-level) for Kubernetes debugging, not `ctr`. This is the single most common source of "my images are missing" confusion.
- **Snapshotter choice matters more than the daemon.** overlayfs is the safe default. `devmapper` requires block-device setup and is easy to misconfigure into disk exhaustion; lazy-pull snapshotters (stargz, SOCI) can dramatically cut cold-start image pull time for large images but add moving parts. Switching snapshotters is not a live migration — existing snapshots are not portable.
- **Config version churn.** The `/etc/containerd/config.toml` format changed across the 1.x line, and again with 2.0 (which raised the config to version 3 and removed long-deprecated fields and plugin names)[^6]. A config that worked on 1.6 may fail to parse on 2.x. Always regenerate with `containerd config default` after a major upgrade rather than carrying the old file forward.
- **2.0 removed deprecated features outright.** The v1 CRI runtime `RuntimeClass` handling, the old `io.containerd.runtime.v1.linux` runtime, and Docker Schema 1 image pulls were dropped[^6]. Nodes relying on ancient images or config can break on upgrade.
- **Pause image and sandbox.** Under CRI, every pod has a "sandbox" (pause) container; a stale or unreachable `sandbox_image` reference (registry auth, air-gapped mirror) breaks pod creation with errors that don't obviously point at containerd.
- **1.6 was the long-term-support anchor** for the dockershim-removal era; many clusters standardized on it before moving to 1.7 and 2.x. Track the RELEASES.md support matrix rather than assuming the newest branch is what your distro ships[^7].
- **It is rarely the root cause.** Because containerd sits below Docker/Kubernetes, most "containerd" incidents trace to runc, the kernel (cgroup v2, overlayfs), CNI, or the registry. Read the shim and kernel logs, not just the daemon.

## When to Use / When Not

**Use when:**
- You are building a platform (Kubernetes distro, PaaS, CI runner, edge orchestrator) and need a stable, embeddable runtime with a clean gRPC API.
- You want the same runtime Kubernetes and Docker already standardize on, with a wide compatibility surface.
- You need pluggable snapshotters or custom runtimes (gVisor, Kata) behind a uniform interface.

**Avoid / don't reach for it when:**
- You want a developer-facing container tool — use Docker or nerdctl/Podman; containerd's own CLIs are debugging tools.
- You need to *build* images — containerd doesn't; use BuildKit, Buildah, or Docker.
- You want daemonless, rootless-first local workflows — Podman is a better fit.
- You only run Kubernetes and want the smallest possible runtime — CRI-O is purpose-built for exactly that scope.

## Alternatives

- cri-o/cri-o — use instead when your only consumer is Kubernetes and you want a runtime scoped to CRI with nothing extra.
- containers/podman — use instead for daemonless, rootless local/dev workflows and a Docker-compatible CLI without a background service.
- moby/moby (Docker Engine) — the full build-and-run developer product; note it runs *on top of* containerd rather than replacing it.
- opencontainers/runc — the low-level OCI runtime containerd drives; a layer below, not a substitute, but relevant when isolation/runtime bugs are actually there.
- kata-containers/kata-containers — use instead (or alongside, as a containerd runtime shim) when you need VM-strength isolation per container.

## History

| Version | Date | Notes |
|---------|------|-------|
| (extracted) | 2016 | Split out of the Docker Engine as a standalone runtime. |
| donated | 2017-03 | Contributed to the CNCF[^1]. |
| 1.0 | 2017-12 | First GA release; stable gRPC API[^7]. |
| 1.1 | 2018-04 | CRI plugin built in and enabled by default[^4]. |
| graduated | 2019-02 | Reached CNCF graduated status[^2]. |
| 1.6 | 2022-02 | Long-term-support line during the Kubernetes dockershim removal[^3]. |
| 1.7 | 2023-03 | Transfer service, NRI, and sandbox API additions. |
| 2.0 | 2024-11 | First major since 1.0: config v3, Go module `/v2`, removal of deprecated runtimes/features[^6]. |

## References

[^1]: CNCF, "containerd Joins the Cloud Native Computing Foundation" — 2017. https://www.cncf.io/announcements/2017/03/29/containerd-joins-cloud-native-computing-foundation/
[^2]: CNCF, "containerd Project Graduation" — 2019-02-28. https://www.cncf.io/announcements/2019/02/28/cncf-announces-containerd-graduation/
[^3]: Kubernetes blog, "Dockershim Removal" (Kubernetes 1.24). https://kubernetes.io/blog/2022/02/17/dockershim-faq/
[^4]: containerd README. https://github.com/containerd/containerd/blob/main/README.md
[^5]: containerd runtime v2 (shim) documentation. https://github.com/containerd/containerd/blob/main/core/runtime/v2/README.md
[^6]: containerd 2.0 release notes. https://github.com/containerd/containerd/releases/tag/v2.0.0
[^7]: containerd RELEASES.md — versioning and support matrix. https://github.com/containerd/containerd/blob/main/RELEASES.md

## Tags

go, container-runtime, kubernetes, cri, oci, docker, cncf, containers, runc, daemon, infrastructure
