# containerd/nerdctl

> A Docker-compatible command-line interface for containerd, exposing containerd's newer features behind familiar `docker`-style commands.

[GitHub repo](https://github.com/containerd/nerdctl) ·
[License: Apache-2.0](https://github.com/containerd/nerdctl/blob/main/LICENSE)

## Overview

nerdctl ("contai**nerd** ctl") is a CLI that speaks the Docker command grammar
(`run`, `build`, `compose`, `ps`, `logs`, `save`, `load`) but drives
[containerd](https://containerd.io) directly instead of the Docker daemon[^1]. It
is a **non-core sub-project** of containerd, started in 2020 by Akihiro Suda, a
containerd/Moby maintainer who also authored RootlessKit, bypass4netns, and much
of the rootless-containers stack[^2]. The stated goal is not to compete with
Docker but to give users a way to exercise containerd features that Docker has
not yet surfaced — lazy image pulling, image encryption, IPFS distribution,
cosign signing, and low-overhead rootless networking.

The defining tension is that nerdctl is a thin, stateless client over a stack of
independently versioned daemons. There is no `nerdctld` — the CLI talks to
containerd's gRPC socket, shells out to BuildKit for builds, to CNI plugins for
networking, and to RootlessKit for rootless mode. That layering is what makes
nerdctl small and fast to iterate, but it also means "install nerdctl" is really
"assemble and keep in sync five moving parts," and behavior depends on the
versions of components nerdctl does not itself ship.

nerdctl is the CLI most users of containerd-native environments actually touch:
Lima and Rancher Desktop bundle it as their default container tool on macOS and
Windows[^3], and it is a common way to poke at the `k8s.io` containerd namespace
on Kubernetes nodes.

## Getting Started

```bash
# Full distribution bundles containerd, CNI, BuildKit, RootlessKit
tar Cxzvvf /usr/local nerdctl-full-<VERSION>-linux-amd64.tar.gz
systemctl --user enable --now containerd   # rootless
```

```console
# Docker-identical basic flow
$ nerdctl run -it --rm alpine

# Build with BuildKit (buildkitd must be running)
$ nerdctl build -t foo .

# Compose, reimplemented in Go — no docker-compose binary needed
$ nerdctl compose -f docker-compose.yaml up -d

# Inspect a Kubernetes node's containers
$ nerdctl --namespace k8s.io ps -a
```

The `-full` tarball ships the whole dependency set; the plain `nerdctl-<ver>`
tarball ships only the CLI and assumes containerd, CNI, and BuildKit already
exist. On macOS the supported path is Lima (`lima nerdctl ...`); on Windows,
Scoop with WSL2 for Linux containers.

## Architecture / How It Works

nerdctl is a translation layer, not a runtime. Each subcommand maps Docker
semantics onto containerd primitives:

- **State** lives in containerd (containers, images, snapshots) plus a small
  metadata store under `/var/lib/nerdctl` for Docker-shaped concepts containerd
  has no native notion of — networks, volumes, and container labels[^4]. This is
  why `nerdctl ps` shows containers nerdctl created but not necessarily those
  created by `ctr` or the kubelet's CRI plugin.
- **Namespaces** are containerd namespaces, not Docker's. The default is
  `default`; Kubernetes containers all live in `k8s.io` regardless of their
  Kubernetes namespace. Objects in different containerd namespaces are invisible
  to each other, a frequent source of "my container disappeared" confusion.
- **Builds** are delegated to BuildKit (`buildkitd`), which must be running as a
  separate daemon. There is no built-in builder.
- **Networking** uses CNI plugins (bridge by default, 10.4.0.0/24). Port
  forwarding and multi-network attach are implemented on top of CNI.
- **Compose** is a from-scratch Go reimplementation of the Compose Spec, not a
  wrapper around the Python/Go `docker compose` — parity is high but not total.
- **Rootless mode** runs containerd inside a RootlessKit network/user namespace.
  Default networking (slirp4netns) is slow; `bypass4netns` restores near-native
  throughput at the cost of an opt-in annotation.

Snapshotters are the headline differentiator: `--snapshotter=stargz|nydus|
overlaybd|soci` enable lazy (on-demand) image pulling, so a container can start
before its full image is transferred. These are containerd features nerdctl
merely exposes.

## Production Notes

- **nerdctl is not a daemon and not an orchestrator.** It manages one host. There
  is no `nerdctl swarm`, no clustering, no restart supervision beyond what
  containerd and `--restart=always` provide. Treat it as `docker` for a single
  node, not as a control plane.
- **Dependency drift is the main operational cost.** BuildKit, CNI, RootlessKit,
  and containerd all version independently. The README itself flags version
  floors (e.g. `nerdctl system prune` needs BuildKit ≥ 0.11)[^1]. Using the
  `-full` tarball or a distribution (Rancher Desktop, Lima) is how most people
  avoid the assembly problem.
- **Namespace footgun.** Forgetting `--namespace k8s.io` on a Kubernetes node
  makes it look like nothing is running. Conversely, creating containers in
  `default` and expecting the kubelet to see them will not work.
- **Cross-tool visibility is partial.** Because volume/network/label metadata is
  nerdctl-specific, containers created by other containerd clients show up with
  missing fields, and vice versa. nerdctl and `ctr` share containerd but not each
  other's higher-level bookkeeping.
- **Rootless networking.** Out of the box slirp4netns caps throughput; production
  rootless deployments should enable bypass4netns. Rootless setup is a scripted
  step (`containerd-rootless-setuptool.sh install`), not automatic.
- **Compose parity gaps.** Most Compose files run unmodified, but exotic keys or
  Docker-daemon-specific behaviors can differ; test critical stacks before
  migrating off `docker compose`.
- **Windows containers are experimental**; Linux-on-WSL2 is the tested Windows
  path.

## When to Use / When Not

**Use when:**
- You run containerd directly (Kubernetes nodes, Rancher Desktop, Lima) and want
  a `docker`-shaped CLI over it.
- You need containerd-only features: lazy pulling (Stargz/Nydus/SOCI), ocicrypt
  image encryption, cosign verify-on-pull, or low-overhead rootless.
- You want to debug or inspect the `k8s.io` namespace on a node without the
  awkwardness of `ctr` or `crictl`.

**Avoid when:**
- You want a single self-contained install with a supported daemon and the full
  ecosystem (registry, buildx, swarm) — Docker/Moby is less to assemble.
- You need multi-host orchestration — this is a single-node tool; use Kubernetes.
- You want maximum command-for-command Docker compatibility with zero surprises;
  nerdctl covers the common surface but not every flag.

## Alternatives

- moby/moby — the Docker Engine; use it when you want one batteries-included
  daemon and the broadest tooling/ecosystem compatibility.
- containers/podman — daemonless Docker-alternative with pods and mature
  rootless; use it when you are not committed to the containerd runtime.
- containerd/containerd — its bundled `ctr` is the lowest-level client; use when
  you need raw containerd access and don't want Docker UX.
- kubernetes-sigs/cri-tools — `crictl` speaks CRI directly; use it to debug the
  kubelet's exact view of a node rather than a Docker-shaped one.
- abiosoft/colima or lima-vm/lima — use these to *get* nerdctl on macOS rather
  than as competitors; they package the containerd+nerdctl stack in a VM.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-12 | Project started as a containerd non-core sub-project[^2]. |
| 0.x | 2021 | Rapid early iteration; Compose, rootless, snapshotters added. |
| 1.0.0 | 2022-10 | First GA / stable release[^5]. |
| 2.0.0 | 2024-11 | Second major line; Go module path `.../nerdctl/v2`[^6]. |

## References

[^1]: nerdctl README — "Docker-compatible CLI for containerd." https://github.com/containerd/nerdctl/blob/main/README.md
[^2]: containerd non-core sub-project governance and maintainers. https://github.com/containerd/project/blob/main/GOVERNANCE.md
[^3]: Lima integrates nerdctl for macOS; Rancher Desktop ships containerd+nerdctl. https://github.com/lima-vm/lima
[^4]: nerdctl directory layout (`/var/lib/nerdctl`). https://github.com/containerd/nerdctl/blob/main/docs/dir.md
[^5]: nerdctl releases page. https://github.com/containerd/nerdctl/releases
[^6]: nerdctl v2 Go module path (`github.com/containerd/nerdctl/v2`), per go.mod. https://github.com/containerd/nerdctl/blob/main/go.mod

## Tags

go, containerd, containers, docker-compatible, cli, oci, rootless, kubernetes, buildkit, container-runtime
