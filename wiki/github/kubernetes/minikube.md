# kubernetes/minikube

> A local Kubernetes cluster that runs on macOS, Linux, and Windows — the reference tool for "one command, one node, real kubeadm."

[GitHub repo](https://github.com/kubernetes/minikube) ·
[Official website](https://minikube.sigs.k8s.io/) ·
[License: Apache-2.0](https://github.com/kubernetes/minikube/blob/master/LICENSE)

## Overview

minikube provisions a throwaway single-node (optionally multi-node) Kubernetes
cluster on a developer laptop or a CI runner. It is a Kubernetes SIG
Cluster-Lifecycle project and has existed since 2016, making it one of the
oldest local-Kubernetes tools still in active development[^1]. Its stated goals
are to be the best tool for *local application development* and to support as
many upstream Kubernetes features as practically fit into a local environment —
LoadBalancer services, ingress, persistent volumes, GPU passthrough, a
dashboard, and a curated addon marketplace[^2].

The defining design choice is the **driver abstraction**: the same `minikube
start` command can target a full VM (hyperkit, kvm2, virtualbox, qemu, hyperv,
vmware), a container (docker, podman), or the bare host (`none`/`ssh`). Inside
that isolation boundary minikube boots real upstream Kubernetes via `kubeadm`,
so the cluster behaves like a production one rather than an emulation. That
fidelity is minikube's differentiator and also its cost: it is heavier and
slower to start than container-only tools like kind or k3d, because "run real
Kubernetes in a real isolation layer" is inherently more work than "run kubelet
in a container."

The project is broadly used and continuously maintained — tens of thousands of
stars, thousands of forks, releases roughly monthly, and commits landing daily.
It is a safe default for teaching, demos, and single-developer workflows, less
so for high-density CI where startup time and resource footprint dominate.

## Getting Started

```bash
# macOS (Homebrew); see the docs for Linux/Windows installers
brew install minikube

# Start a cluster using the auto-detected driver (Docker if available)
minikube start

# kubectl is configured automatically to point at the new cluster
kubectl get nodes
minikube dashboard        # opens the web UI
minikube addons enable ingress
minikube delete           # tear it all down
```

```bash
# Pin a driver and Kubernetes version explicitly (reproducible)
minikube start --driver=docker --kubernetes-version=v1.30.0 --cpus=4 --memory=8g
```

## Architecture / How It Works

A `minikube start` runs, roughly, this pipeline: pick a **driver**, create or
reuse an isolation target (a VM, a container, or the host), copy in the
`kubeadm`/`kubelet` binaries and a container runtime, then run `kubeadm init`
inside it and write a matching context into `~/.kube/config`.

Two isolation strategies coexist:

- **VM drivers** boot minikube's own Buildroot-based `minikube.iso`, a small
  Linux image carrying the container runtime and Kubernetes components. This is
  the original path and the most isolated.
- **KIC drivers** (`docker`, `podman`) skip the VM and run everything inside a
  single privileged container built from the **kicbase** image — "Kubernetes in
  container," similar in spirit to kind[^3]. On macOS and Windows this container
  itself lives inside the Docker Desktop VM, so you get *nested* virtualization
  whose networking has real consequences (see Production Notes). The Docker
  driver is the auto-selected default on most machines today.

**Container runtime** is pluggable via `--container-runtime`: containerd,
CRI-O, or Docker (dockershim-free, through cri-dockerd). The runtime choice
changes how you get images into the cluster — this is the single most common
source of "it works on my host but the pod says ImagePullBackOff."

**Profiles** (`-p <name>`) give each cluster its own isolated state directory
under `~/.minikube/profiles`, which is how multi-cluster and multi-node
(`--nodes N`) work. **Addons** are versioned YAML manifests minikube applies and
reconciles on your behalf (ingress-nginx, metrics-server, registry, dashboard,
storage-provisioner, gvisor, and more) — convenient, but they pin their own
component versions independently of your cluster version.

## Production Notes

minikube is a *development* tool; "production" here means the operator caveats
that bite in daily and CI use.

- **Images built on the host are invisible to the cluster.** The cluster has its
  own image store. Use `minikube image load <img>` (all runtimes),
  `minikube image build`, or point your build at the in-VM Docker daemon with
  `eval $(minikube docker-env)` — but `docker-env` only works with the Docker
  runtime, not containerd/CRI-O. Forgetting this produces `ImagePullBackOff` on
  images that exist locally. Setting `imagePullPolicy: IfNotPresent` is usually
  required for locally-loaded images.
- **LoadBalancer needs `minikube tunnel`.** `type: LoadBalancer` services stay
  `<pending>` until you run `minikube tunnel` in a separate terminal, which
  needs root/sudo to bind privileged ports and route to cluster IPs. On the
  Docker driver on macOS/Windows the node IP is also unreachable from the host
  without the tunnel or `minikube service <svc>`.
- **Docker-driver networking is nested and non-obvious.** Because the cluster
  runs inside the Docker Desktop VM, `NodePort` and node IPs are not directly
  routable from the host OS; `minikube service --url` prints a working host URL
  by opening a proxy. Ingress on the Docker driver on macOS likewise requires
  the tunnel.
- **Resource footprint.** A default cluster reserves ~2 CPU / 2–6 GB RAM and
  boots in tens of seconds to minutes depending on driver. For per-test
  ephemeral clusters at scale, kind and k3d start faster and lighter; teams
  running many parallel CI clusters often prefer those.
- **The `none` driver runs on the host as root**, with no isolation — historically
  popular for CI because it avoids nested virt, but it mutates the host, can
  conflict with an existing container runtime, and is a poor fit for shared
  machines.
- **Upgrade / version skew.** minikube bundles a default Kubernetes version per
  release; upgrading minikube can change the cluster version under you unless you
  pin `--kubernetes-version`. Profiles created by an older minikube may not be
  cleanly reusable after a major minikube upgrade — `minikube delete` (or
  `--all`) and recreate is the reliable path, and `~/.minikube` occasionally
  needs a manual wipe after failed starts.
- **Apple Silicon**: hyperkit and virtualbox are unavailable; use the `docker`
  or `qemu` driver.

## When to Use / When Not

**Use when:**
- You want a real upstream Kubernetes cluster locally with the widest feature
  coverage (ingress, LoadBalancer via tunnel, PVs, GPU, dashboard, addons).
- You are teaching, demoing, or doing single-developer app development and value
  a batteries-included, cross-platform, driver-flexible tool.
- You need a container runtime other than the default, or need to exercise
  behavior that only a VM-isolated node reproduces.

**Avoid when:**
- You run many ephemeral clusters in CI and startup time / density dominate —
  kind or k3d are leaner.
- You want the absolute lightest footprint or edge-oriented distro — k3s/k3d.
- You are on a locked-down machine where nested virtualization or a privileged
  container is not permitted.

## Alternatives

- kubernetes-sigs/kind — Kubernetes-in-Docker; use instead when you want the
  fastest, lightest per-test clusters in CI and don't need VM isolation or addons.
- k3d-io/k3d — runs k3s in Docker; use when you want a minimal, fast cluster and
  are fine with the k3s distro's trimmed defaults.
- k3s-io/k3s — a lightweight single-binary Kubernetes distro; use for edge/IoT or
  small production, not just local dev.
- canonical/microk8s — snap-based single-node/HA Kubernetes; use on Ubuntu/Linux
  when you want an addon-driven distro managed via snap.
- Docker Desktop Kubernetes — use when you already run Docker Desktop and want a
  one-checkbox cluster with no extra tooling, accepting less control.

## History

| Version | Date | Notes |
|---------|------|-------|
| Project start | 2016-04 | Initial repo under the Kubernetes org[^1]. |
| 1.0.0 | 2019-03 | First stable 1.x release; VM/ISO drivers, kubeadm bootstrap. |
| 1.9.x | 2020-04 | Docker (KIC) driver introduced via the kicbase image[^3]. |
| 1.10.x | 2020 | Multi-node cluster support (`--nodes`). |
| 1.x (ongoing) | monthly | Docker driver becomes the common default; containerd/CRI-O runtimes, GPU addons, growing addon marketplace[^2]. |

*(Exact per-version dates beyond the above are omitted where not verified;
consult the GitHub releases page for the authoritative changelog.)*

## References

[^1]: kubernetes/minikube on GitHub — repository created 2016. https://github.com/kubernetes/minikube
[^2]: minikube documentation — features, addons, and handbook. https://minikube.sigs.k8s.io/docs/
[^3]: minikube "kicbase" / Docker driver documentation. https://minikube.sigs.k8s.io/docs/drivers/docker/

## Tags

go, kubernetes, local-development, containers, cncf, developer-tools, devops, cli, cluster, cri
