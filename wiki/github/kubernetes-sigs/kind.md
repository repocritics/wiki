# kubernetes-sigs/kind

> Run local Kubernetes clusters using Docker containers as the "nodes" — built first to test Kubernetes itself.

[GitHub repo](https://github.com/kubernetes-sigs/kind) ·
[Official website](https://kind.sigs.k8s.io/) ·
[License: Apache-2.0](https://github.com/kubernetes-sigs/kind/blob/main/LICENSE)

## Overview

kind ("Kubernetes IN Docker") runs a full Kubernetes cluster where each node — control plane or worker — is a Docker (or Podman/nerdctl) container instead of a VM or physical host. Inside each container runs systemd, a container runtime (containerd), and the standard kubelet/kubeadm stack, so the cluster is a real, conformant Kubernetes install rather than an emulation. It is a Kubernetes SIG-Testing project, originally written by Ben Elder to give Kubernetes' own CI a fast, reproducible way to stand up clusters for end-to-end tests[^1].

That origin explains its defining tradeoff. kind optimizes for speed, reproducibility, and CI-friendliness: a single-node cluster comes up in tens of seconds, tears down cleanly, and needs nothing but a container engine. In exchange it inherits the container model's rough edges — the "nodes" share the host kernel, LoadBalancer services and node IPs are not reachable from the host by default, and image/config changes usually mean recreating the cluster rather than mutating it. It is deliberately not a production Kubernetes distribution and does not try to be one.

kind is a CNCF-certified conformant Kubernetes installer and one of the three common local-cluster tools alongside minikube and k3d. It has never shipped a 1.0 — the version series is still `0.x`, and a 1.0 roadmap remains open[^2]. In practice the `0.x` label understates its stability; it is a core dependency of Kubernetes' own test infrastructure.

## Getting Started

```bash
# Homebrew (macOS/Linux)
brew install kind
# or Go toolchain
go install sigs.k8s.io/kind@v0.32.0
# or download a static binary from the releases page
```

```bash
kind create cluster                 # single-node cluster named "kind"
kubectl get nodes                   # kubeconfig context is "kind-kind"
kind delete cluster
```

A multi-node cluster with host port mappings, via a config file:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080      # nodePort -> host
        hostPort: 8080
  - role: worker
  - role: worker
```

```bash
kind create cluster --name dev --config kind-config.yaml
kind load docker-image myapp:latest --name dev   # push a local image into the nodes
```

## Architecture / How It Works

kind's core loop is: pull a prebuilt **node image** (`kindest/node`, tagged by Kubernetes version), start one container per node, and run **kubeadm** inside them to bootstrap the control plane and join workers[^3].

- **Node image.** `kindest/node` is a Docker image containing systemd as PID 1, containerd, the kubelet, kubeadm, CNI binaries, and a set of preloaded Kubernetes component images for a specific K8s version. Because the K8s version is baked into the image, "upgrading Kubernetes" means selecting a different node image and recreating the cluster — there is no in-place upgrade path.
- **Container runtime.** The host engine (Docker/Podman/nerdctl) runs the node *containers*; inside each node, **containerd** is the CRI that runs the actual pods. This is nested containerization, not Docker-in-Docker in the classic sense, and it requires privileged containers plus a writable cgroup hierarchy.
- **Networking.** The default CNI is **kindnet**, a minimal plugin sufficient for most testing. It can be disabled (`disableDefaultCNI: true`) to install Calico, Cilium, etc. All nodes share a Docker network; pod and service CIDRs live inside it.
- **Storage.** The default StorageClass is backed by Rancher's local-path-provisioner, writing PersistentVolumes to the node container's filesystem — ephemeral by design.
- **kubeconfig.** kind merges credentials into `~/.kube/config` under a `kind-<name>` context and can also export them with `kind get kubeconfig`.

The `pkg/cluster` and `pkg/build` Go packages implement cluster lifecycle and node-image builds, and the `kind` CLI is a thin wrapper over them. Building node images from Kubernetes source (`kind build node-image`) is what ties kind back to its CI-testing purpose: it lets Kubernetes developers test arbitrary commits of Kubernetes on a throwaway cluster.

## Production Notes

kind is a testing/development tool; "production notes" here means operator footguns, most of which stem from the container-as-node model.

- **Nodes aren't reachable from the host by default.** On Docker Desktop (macOS/Windows) the engine runs in a VM, so node and pod IPs are not routable from your laptop. Use `extraPortMappings` + a NodePort, `kubectl port-forward`, or an ingress controller with host ports rather than expecting to `curl` a pod IP.
- **LoadBalancer services stay `<pending>`.** There is no cloud controller. Use `cloud-provider-kind` or MetalLB to get external IPs, or fall back to NodePort/port-forward.
- **inotify / file-descriptor limits.** Multi-node clusters frequently exhaust `fs.inotify.max_user_watches` and `max_user_instances`, showing up as pods stuck in crash loops or "too many open files". Raising the host sysctls is the standard fix and is documented as a known issue[^4].
- **Image loading is explicit.** Nodes do not see your local Docker daemon's images. You must `kind load docker-image` (or `kind load image-archive`) into the cluster, or push to a registry the nodes can reach; forgetting this yields `ImagePullBackOff` for images that "exist locally".
- **Resource pressure.** Each node is a container running systemd + containerd + kubelet; multi-node HA clusters can consume several GB of RAM. Docker Desktop's default memory limit is a common bottleneck.
- **Ephemerality.** Clusters generally do not survive a Docker daemon or host restart intact; treat them as disposable. Persist nothing you can't recreate.
- **cgroup v2 / rootless Podman.** Modern hosts need cgroup v2 delegation configured correctly; rootless Podman adds further constraints and is more fragile than the Docker path.

## When to Use / When Not

**Use when:**
- You need ephemeral, reproducible clusters in CI — this is kind's home turf.
- You want multi-node or HA control-plane topologies locally without VMs.
- You're testing Kubernetes itself, a CNI, an operator, or CRDs against a real, conformant API server.
- You want the lightest cluster that is still "really Kubernetes."

**Avoid when:**
- You want a long-lived local cluster with add-ons and a dashboard — minikube's addon system fits better.
- You want the absolute lightest/fastest single-node dev loop — k3d (k3s) starts faster and uses less memory.
- You need production Kubernetes — kind is explicitly not for production; use a real distro.
- You need LoadBalancer/ingress to "just work" from the host with no extra wiring.

## Alternatives

- rancher/k3d — wraps k3s in Docker; lighter and faster to start. Use instead when memory footprint and startup time matter more than matching upstream Kubernetes exactly.
- kubernetes/minikube — VM or Docker driver with a rich addon ecosystem. Use instead when you want a persistent local cluster with batteries-included ingress, registry, and dashboard.
- k3s-io/k3s — a lightweight but production-capable distribution. Use instead when the cluster should outlive a dev session or run at the edge.
- canonical/microk8s — snap-based single-command cluster. Use instead on Ubuntu hosts wanting an OS-integrated install.
- Docker Desktop's built-in Kubernetes — one toggle, single node. Use instead for the simplest possible local API server when you already run Docker Desktop.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2018-09 | SIG-Testing project started by Ben Elder for Kubernetes CI[^1]. |
| 0.1.0 | 2019 | First tagged release; kubeadm-based node bootstrap. |
| 0.x series | 2019–2026 | Podman/nerdctl support, containerd runtime, kindnet, multi-node/HA, config `v1alpha4`. |
| 0.32.0 | 2026 | Current release referenced by the README[^5]. |

*Per-version release dates above are approximate and were not verified against a release API for this page — see the releases page for exact tags.*

## References

[^1]: kind — "Kubernetes IN Docker", SIG-Testing. README and project overview. https://github.com/kubernetes-sigs/kind
[^2]: kind 1.0 roadmap. https://kind.sigs.k8s.io/docs/contributing/1.0-roadmap
[^3]: kind design documentation (initial design). https://kind.sigs.k8s.io/docs/design/initial
[^4]: kind — "Known Issues" (inotify limits, Docker resources, etc). https://kind.sigs.k8s.io/docs/user/known-issues/
[^5]: kind quick start / installation. https://kind.sigs.k8s.io/docs/user/quick-start/

## Tags

kubernetes, docker, containers, local-development, testing, ci, go, kubeadm, containerd, cluster-management, podman, devtools
