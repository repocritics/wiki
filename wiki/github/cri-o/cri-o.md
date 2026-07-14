# cri-o/cri-o

> A Kubernetes-only container runtime — implements the Container Runtime Interface and nothing more, delegating execution to an OCI runtime.

[GitHub repo](https://github.com/cri-o/cri-o) ·
[Official website](https://cri-o.io) ·
[License: Apache-2.0](https://github.com/cri-o/cri-o/blob/main/LICENSE)

## Overview

CRI-O is a container runtime whose entire scope is to be the glue between the
Kubernetes kubelet and an OCI-conformant runtime such as `runc` or `crun`[^1]. It
implements the kubelet's gRPC Container Runtime Interface (CRI) and delegates the
actual container execution, image handling, storage, and networking to purpose-built
libraries. The stated design goal is deliberate minimalism: it is *not* a
general-purpose container engine and ships no supported end-user CLI[^2].

The project began as "ocid" in 2016, led largely by Red Hat engineers, as a direct
response to Kubernetes needing a runtime that tracked the CRI rather than the whole
Docker feature surface. Version 1.0 shipped in October 2017, and CRI-O was accepted
into the CNCF as an incubating project in 2019. Its defining constraint is that
minor versions are pinned to Kubernetes minor versions — CRI-O `1.x` targets
Kubernetes `1.x` — which follows the Kubernetes `n-2` version-skew policy[^3].

That version coupling is CRI-O's whole personality: it trades the flexibility of a
standalone runtime for tight, predictable alignment with a single orchestrator. It
is the default runtime in Red Hat OpenShift, which is the main reason the project is
well-funded and heavily production-tested, but also why its center of gravity sits
in the Red Hat / OKD ecosystem rather than the broader managed-Kubernetes market,
where containerd is more common.

## Getting Started

CRI-O is installed as a system daemon on each Kubernetes node, at a version matching
the cluster's Kubernetes minor version. Packages live in the separate
`cri-o/packaging` repository:

```bash
# Install the CRI-O stream matching your Kubernetes minor version (example: 1.30)
# then enable the daemon
sudo systemctl enable --now crio
sudo systemctl status crio
```

You do not drive CRI-O directly; the kubelet does, over its socket. For debugging,
use `crictl` (the CRI-level CLI from cri-tools):

```bash
# Talk to the running CRI-O socket
sudo crictl --runtime-endpoint unix:///var/run/crio/crio.sock ps
sudo crictl images
```

Configuration is TOML, with drop-in fragments preferred over editing the main file:

```toml
# /etc/crio/crio.conf.d/10-runtime.conf
[crio.runtime]
default_runtime = "crun"
conmon_cgroup   = "pod"
cgroup_manager  = "systemd"
```

## Architecture / How It Works

CRI-O is a gRPC server listening on `/var/run/crio/crio.sock`, implementing the CRI
service the kubelet expects. It owns no execution logic of its own; instead it
orchestrates a set of `containers/*` and OCI components:

- **OCI runtime** — `runc` (default) or `crun`. CRI-O writes an OCI runtime spec and
  invokes the runtime binary to create/start/kill containers. Multiple runtimes can
  be registered and selected per-workload via `RuntimeClass`.
- **conmon** — a small C monitor process spawned *per container*. It owns the
  container's PTY, streams logs, and records the exit code. Because conmon is the
  parent, CRI-O itself can be restarted or upgraded without killing running
  containers[^4]. `conmon-rs`, a Rust rewrite that monitors at the pod level, is the
  newer alternative.
- **containers/storage** — image layer storage and the overlay filesystem (`overlay`
  is the default driver; `btrfs`, `vfs` and others exist).
- **containers/image** — image pulls, signature verification (`policy.json`) and
  registry configuration (`registries.conf`), shared with Podman and Buildah.
- **CNI** — pod networking via the standard CNI plugins.

Because storage and image handling are the same libraries Podman and Buildah use, a
node running CRI-O shares on-disk image format and configuration semantics with those
tools — a real advantage when debugging. The runtime also exposes an unstable HTTP
status API (`crio status info`) for introspection, explicitly documented as not for
production dependence.

## Production Notes

**Version pinning is not optional.** The most common operational failure is running
a CRI-O minor version that does not match the kubelet's. Upgrades must move CRI-O and
Kubernetes together within the supported skew; automation that upgrades one without
the other will produce a node that fails to register or start pods.

**Package repo migration.** Installation historically ran through the openSUSE Build
Service (OBS). The package hosting was reorganized into the `cri-o/packaging` repo
and new repository URLs; older install docs and pipelines pointing at the retired OBS
paths break. Confirm you are using current package sources before debugging a "works
on my node" install failure.

**cgroup driver must match.** `cgroup_manager` in CRI-O and `cgroupDriver` in the
kubelet must both be `systemd` (the modern default) or both `cgroupfs`. A mismatch
leads to pods that start but are mis-accounted or killed under memory pressure.

**crun vs runc.** `runc` is the default, but `crun` (C, not Go) uses less memory and
starts faster, and is the default in some OpenShift/Fedora configurations. Switching
is a config change, but validate seccomp/AppArmor and any runtime-specific annotations
after the swap.

**No general-purpose tooling.** There is intentionally no `docker`-like CLI. For any
node-level container inspection you rely on `crictl` (CRI semantics, pod/sandbox
oriented) or Podman against the shared storage — not on CRI-O itself. Teams migrating
from Docker-based nodes routinely underestimate this.

**Debugging surface.** Logs go through conmon to the kubelet's log path; the HTTP
status API and `crio status` help, but deep issues often require reading the OCI
runtime's own state under `/run` and the storage layout under
`/var/lib/containers/storage`.

## When to Use / When Not

**Use when:**
- Your workload is Kubernetes and only Kubernetes; you want a runtime scoped to exactly
  the CRI with a small attack and maintenance surface.
- You run OpenShift/OKD, where CRI-O is the supported, default, and best-tested path.
- You want per-node runtime version alignment with Kubernetes to be explicit and enforced.
- You value sharing image/storage semantics with Podman and Buildah on the same host.

**Avoid when:**
- You need to run containers outside Kubernetes (local builds, CI, ad-hoc runs) — CRI-O
  has no story for that; use Podman or containerd/nerdctl.
- You are on a managed platform (GKE/EKS/AKS) where containerd is the vendor default and
  supported path; swapping brings little benefit and support friction.
- You want a single runtime spanning Kubernetes and general container tooling.

## Alternatives

- containerd/containerd — the more widely deployed CRI runtime and the default in most
  managed Kubernetes; use it when you want the mainstream, multi-consumer runtime.
- opencontainers/runc — the low-level OCI runtime CRI-O actually calls; not a CRI
  implementation, so it is a dependency, not a drop-in replacement.
- containers/crun — faster, lower-memory OCI runtime; use it *under* CRI-O in place of
  runc, not instead of CRI-O.
- kata-containers/kata-containers — VM-isolated OCI runtime; plug it in under CRI-O when
  you need stronger workload isolation than namespaces provide.
- google/gvisor — application-kernel sandbox (`runsc`); use as the OCI runtime under
  CRI-O when you want syscall-level isolation for untrusted workloads.

## History

| Version | Date | Notes |
|---------|------|-------|
| ocid | 2016 | Project started as "ocid" — a CRI runtime for Kubernetes[^1]. |
| 1.0.0 | 2017-10 | First stable release, tracking Kubernetes 1.x[^2]. |
| — | 2019 | Accepted into the CNCF as an incubating project. |
| conmon-rs | ~2022 | Rust pod-level monitor introduced as a conmon successor[^4]. |

CRI-O does not follow independent semantic versioning; each `release-1.x` branch is
maintained in lockstep with the corresponding Kubernetes minor release, with patch
releases cut as needed rather than on a fixed cadence[^3].

## References

[^1]: CRI-O project scope and component plan, repository README. https://github.com/cri-o/cri-o#what-is-the-scope-of-this-project
[^2]: CRI-O — "What is not in the scope of this project" (no supported CLI). https://github.com/cri-o/cri-o/blob/main/README.md
[^3]: CRI-O ⬄ Kubernetes compatibility matrix and version-skew policy. https://github.com/cri-o/cri-o#compatibility-matrix-cri-o--kubernetes
[^4]: conmon-rs, the Rust container monitor used by CRI-O. https://github.com/containers/conmon-rs

## Tags

go, kubernetes, container-runtime, cri, oci, runc, crun, cncf, openshift, containers, infrastructure
