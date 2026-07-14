# google/gvisor

> An application kernel written in Go that reimplements the Linux syscall surface in userspace, sandboxing containers without a hypervisor.

[GitHub repo](https://github.com/google/gvisor) ·
[Official website](https://gvisor.dev) ·
[License: Apache-2.0](https://github.com/google/gvisor/blob/master/LICENSE)

## Overview

gVisor is a userspace kernel that sits between a containerized application and the host Linux kernel. Rather than filtering syscalls (seccomp) or running a full guest OS (a VM), it intercepts application syscalls and services them itself, in Go, implementing a Linux-compatible interface[^1]. The project describes its own approach as "implementing Linux by way of Linux": gVisor reimplements the ABI applications expect, but ultimately leans on a small, tightly controlled set of host syscalls to do real work. It originated inside Google, where variants of the technology have run production workloads (App Engine, Cloud Run, Cloud Functions) for years[^2].

The defining tradeoff is security surface versus compatibility and performance. A normal container shares the host kernel directly, so a single kernel vulnerability can mean container escape. gVisor shrinks the host attack surface to roughly the syscalls its Sentry component actually makes, and re-implements everything else in a memory-safe language. The cost is that gVisor is not Linux: it implements a large but incomplete subset of the ~350 Linux syscalls, and syscall-heavy or I/O-heavy workloads pay a measurable overhead because every syscall crosses gVisor's interception boundary instead of trapping straight into the host kernel[^3].

The primary interface is `runsc`, an OCI-compliant runtime that drops into existing Docker and Kubernetes tooling as an alternative to `runc`. This is the pitch: the same container image, the same `docker run`, a much smaller trusted computing base.

## Getting Started

Install the pre-built `runsc` binary (Linux x86_64 or ARM64), then register it with Docker:

```sh
# Download and install runsc + containerd shim (see gvisor.dev for the
# current release URL and checksum verification steps).
sudo runsc install          # writes runtime config into /etc/docker/daemon.json
sudo systemctl reload docker
```

```sh
# Run an untrusted container under the gVisor sandbox instead of runc.
docker run --rm --runtime=runsc hello-world

# Confirm you are inside the Sentry, not the host kernel: the reported
# kernel version comes from gVisor, not the host.
docker run --rm --runtime=runsc alpine uname -a
```

For Kubernetes, `runsc` is wired in via a `RuntimeClass` object and a node-level containerd config (commonly installed with GKE Sandbox or manually via the containerd shim).

## Architecture / How It Works

gVisor is two cooperating processes per sandbox:

- **Sentry** — the application kernel. It receives every syscall the sandboxed app makes, implements filesystem, network, signal, memory, and process semantics itself, and maintains its own view of process/file/socket state. The Sentry runs with an aggressive seccomp filter so that even if it is compromised, the syscalls it can issue against the host are narrowly bounded.
- **Gofer** — a separate process that mediates filesystem access. The Sentry cannot open host files directly; it sends 9P/LISAFS requests to the Gofer, which holds the actual file descriptors. This keeps host filesystem access out of the Sentry's trust boundary.

Syscall interception uses one of two **platforms**. The `ptrace` platform (renamed `systrap` in newer releases) traps guest syscalls using host kernel facilities and is the portable default. The `KVM` platform uses hardware virtualization to run the Sentry as a guest and trap syscalls via VM exits, which can be faster for some workloads but requires `/dev/kvm` and is sensitive to nested-virtualization environments[^4]. Platform choice is the single biggest performance lever.

Networking has its own subsystem: **netstack**, a from-scratch userspace TCP/IP stack written in Go (`pkg/tcpip`). By default gVisor does not use the host network stack at all; packets are handled entirely in netstack. This is a large surface to reimplement and is the source of both gVisor's network isolation story and a class of compatibility gaps (certain socket options, protocols, and offload features). A "host networking" mode exists that bypasses netstack at the cost of isolation.

The whole system is built with **Bazel**, not the Go toolchain. A synthetic `go` branch is published for `go get` compatibility, but it is best-effort; real development happens on `master` under Bazel[^5].

## Production Notes

- **Not every syscall is implemented.** gVisor covers a large subset, but applications that use exotic syscalls, unusual `ioctl`s, or niche `/proc` and `/sys` files can fail in ways that look like kernel bugs. Check the documented compatibility notes before assuming a workload runs unmodified.
- **Syscall and I/O overhead is real.** Because every syscall is intercepted and serviced in userspace, syscall-bound workloads (heavy small-file I/O, fork-heavy, high-connection-churn networking) see the largest slowdowns; CPU-bound compute sees little. Benchmarks vary widely by platform and workload — measure your own; do not assume the marketing "near-native."
- **Filesystem performance depends on the Gofer path.** Overlay/cache modes and the `--overlay2` options materially change file I/O throughput. The Gofer indirection means path-heavy workloads (Python imports, node_modules) can be slow with the wrong caching settings.
- **Platform selection matters and is environment-specific.** KVM needs `/dev/kvm` and behaves poorly under nested virtualization on some clouds; systrap/ptrace is more portable but has different overhead characteristics. This is the first thing to tune.
- **GPU and direct-hardware workloads are limited.** GPU support (via `nvproxy`) exists but is narrower than raw host access; device-passthrough-heavy workloads may not fit.
- **It is a moving target.** gVisor ships frequent releases with a date-based versioning scheme (`YYYYMMDD.p`), not semantic versions, so there is no stable "LTS" line — pin a release and test upgrades.

## When to Use / When Not

**Use when:**
- You run untrusted or multi-tenant code (user-submitted functions, CI runners, notebook backends) and want defense-in-depth beyond namespaces + seccomp.
- You want an OCI runtime that drops into existing Docker/Kubernetes without rewriting images.
- You want a smaller host kernel attack surface without the operational weight of full VMs.

**Avoid when:**
- Your workload is syscall- or I/O-bound and latency-sensitive; the interception overhead may be unacceptable.
- You depend on syscalls, `ioctl`s, kernel modules, or hardware features gVisor does not implement.
- You need the full, exact Linux ABI — gVisor is Linux-compatible, not Linux-identical.

## Alternatives

- kata-containers/kata-containers — use instead when you want VM-grade isolation with a real Linux guest kernel and can afford lightweight-VM overhead and full ABI fidelity.
- firecracker-microvm/firecracker — use instead when the unit of isolation is a microVM (serverless/functions) and you want a minimal VMM rather than a userspace kernel.
- opencontainers/runc — use instead when you trust your workloads and want maximum performance with the standard shared-kernel container model.
- containers/bubblewrap — use instead for lightweight namespace/seccomp sandboxing of desktop or single processes, not multi-tenant container escape hardening.
- weaveworks/ignite — use instead when you want Firecracker microVMs managed with a Docker-like UX.

## History

| Version | Date | Notes |
|---------|------|-------|
| Open-sourced | 2018-05 | Announced and published by Google; `runsc` OCI runtime, Sentry + Gofer design[^1]. |
| netstack / KVM platform | 2018–2019 | Userspace TCP/IP stack and hardware-virtualization platform matured. |
| GKE Sandbox GA | 2019 | gVisor offered as a managed node option on Google Kubernetes Engine[^2]. |
| systrap platform | ~2023 | New default syscall-trap platform succeeding the classic ptrace path[^4]. |
| Ongoing | date-based `YYYYMMDD.p` releases | Frequent rolling releases; no semantic-version LTS line. |

## References

[^1]: gVisor documentation — "What is gVisor?" and architecture guide. https://gvisor.dev/docs/
[^2]: Google Cloud — GKE Sandbox / gVisor usage in Google products. https://cloud.google.com/kubernetes-engine/docs/concepts/sandbox-pods
[^3]: gVisor — Performance Guide (syscall interception overhead, workload sensitivity). https://gvisor.dev/docs/architecture_guide/performance/
[^4]: gVisor — Platforms documentation (ptrace/systrap vs KVM). https://gvisor.dev/docs/architecture_guide/platforms/
[^5]: gVisor README — build via Bazel; best-effort `go` branch. https://github.com/google/gvisor#using-go-get

## Tags

go, container-security, sandbox, kernel, oci-runtime, linux, kubernetes, docker, virtualization, syscall-interception, isolation
