# kubernetes/kubernetes

> Production-grade container orchestration — the declarative control plane that became the industry's default scheduling substrate.

[GitHub repo](https://github.com/kubernetes/kubernetes) ·
[Official website](https://kubernetes.io) ·
[License: Apache-2.0](https://github.com/kubernetes/kubernetes/blob/master/LICENSE)

## Overview

Kubernetes (K8s) is an open-source system for deploying, scaling, and managing containerized applications across a cluster of machines. It was announced by Google in 2014, built on lessons from Google's internal Borg system[^1], and donated as the seed project of the Cloud Native Computing Foundation (CNCF) in 2015. With ~124k stars and ~44k forks it is one of the largest and most-forked projects on GitHub, and its release cadence (roughly three minor versions a year) has stayed steady for a decade — this is infrastructure with institutional backing, not a single-vendor project.

The core idea is **declarative reconciliation**: you describe desired state as API objects (Deployments, Services, ConfigMaps), and a set of controllers continuously works to make observed state match. You do not run imperative "deploy" commands; you `apply` a manifest and controllers converge toward it. This model is Kubernetes' defining strength and its defining tax — it is extremely composable and extensible, but the indirection between "I applied YAML" and "my pod is actually running" is where most operational pain lives.

Kubernetes is written in Go and is deliberately unopinionated about the layers around it: it does not build images, does not provide a service mesh, and ships pluggable interfaces (CRI, CNI, CSI) for the container runtime, networking, and storage rather than one blessed implementation[^2]. That un-opinionatedness is why an entire ecosystem (Istio, Cilium, Prometheus, Argo, cloud-managed control planes) grew on top of it, and why "just install Kubernetes" is never actually just one decision.

## Getting Started

Do not build from source to *use* Kubernetes. For local development, use a single-node distribution:

```bash
# kind — Kubernetes IN Docker (a real cluster inside containers)
kind create cluster --name dev

# or minikube
minikube start
```

A minimal Deployment + Service, applied declaratively:

```yaml
# app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "250m", memory: "256Mi" }
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f app.yaml
kubectl get pods -w          # watch reconciliation
kubectl port-forward svc/web 8080:80
```

Building from source (`git clone … && make`) is only for contributing to Kubernetes itself; using `k8s.io/kubernetes` as a Go library is explicitly unsupported[^3].

## Architecture / How It Works

A cluster is a **control plane** plus a set of **worker nodes**.

Control plane components:

- **kube-apiserver** — the only component that talks to etcd; a REST front end for all API objects, doing authn/authz, admission, and validation. Everything else is a client of the API server.
- **etcd** — the consistent key-value store holding all cluster state. It is the source of truth and the most fragile operational dependency.
- **kube-scheduler** — assigns unscheduled Pods to nodes based on resource requests, affinity/anti-affinity, taints/tolerations, and topology constraints.
- **kube-controller-manager** — runs the built-in control loops (Deployment, ReplicaSet, Node, endpoints, etc.), each watching the API and driving state toward spec.
- **cloud-controller-manager** — isolates cloud-provider-specific logic (load balancers, routes, node lifecycle).

On each node:

- **kubelet** — the node agent; watches for Pods bound to its node and drives the container runtime to run them, reporting status back.
- **container runtime** — via the **Container Runtime Interface (CRI)**; containerd and CRI-O are the common choices since Docker's dockershim was removed in 1.24[^4].
- **kube-proxy** — programs iptables/IPVS (or is replaced by a CNI's own dataplane, e.g. Cilium eBPF) to implement Service virtual IPs.

The unifying pattern is the **controller/reconciler loop**: watch desired state, observe actual state, act to close the gap, repeat. **Custom Resource Definitions (CRDs)** let you add your own API types, and the **Operator pattern** packages a CRD plus a controller to automate stateful software (databases, queues). This extensibility is the reason Kubernetes became a platform-for-building-platforms rather than a fixed product — the same reconciliation machinery that runs Deployments runs cert-manager, Argo CD, and Istio.

Networking assumes a flat model: every Pod gets a routable IP and can reach every other Pod without NAT. The **CNI** plugin implements this; **CSI** does the equivalent for storage volumes. None ship a single default — you choose.

## Production Notes

**etcd is the blast radius.** Almost every catastrophic cluster failure traces back to etcd: slow disks (etcd needs low-latency fsync, SSD-class storage), running out of the default ~8 GB DB quota, or losing quorum during upgrades. Back up etcd, monitor its latency, and treat it as a stateful database, not a config file.

**Resource requests vs. limits are a footgun.** Requests drive scheduling; limits drive enforcement. Omitting CPU/memory requests leads to overcommit and noisy-neighbor evictions; setting memory limits too tight causes OOMKills that look like application bugs. CPU limits cause throttling that many teams later remove. The `Guaranteed`/`Burstable`/`BestEffort` QoS class your Pod lands in — derived implicitly from these values — decides who gets evicted first under pressure.

**Upgrades are frequent and version-skew-constrained.** Kubernetes ships ~3 minor releases/year and each is supported for roughly a year (about 14 months of patches). Skipping versions is not supported — you must upgrade one minor version at a time, and control-plane/kubelet version skew is bounded. Deprecated APIs are removed on a schedule (the `extensions/v1beta1` → `apps/v1` migration and the 1.22 and 1.25 removal waves broke many manifests). `kubectl` and admission webhooks both participate in skew constraints.

**The YAML/indirection tax is real.** A `kubectl apply` that "succeeds" only means the object was accepted, not that the workload is healthy. Debugging spans `kubectl describe`, events (which expire), controller logs, and admission webhooks. `CrashLoopBackOff`, `ImagePullBackOff`, and `Pending` due to unschedulable resource requests are the daily triad.

**Scale limits.** Upstream supports up to ~5,000 nodes and ~150,000 Pods per cluster, but you hit practical limits (etcd size, API server watch load, kube-proxy iptables O(n) rule churn) well before that. Very large fleets usually shard into multiple clusters rather than one giant one.

**You almost certainly want a managed control plane.** Running your own control plane and etcd is real, ongoing operational work. Most teams use EKS, GKE, or AKS and only manage workloads. Self-managed clusters are justified for on-prem, sovereignty, or cost reasons — with the staffing to match.

## When to Use / When Not

**Use when:**
- You run many containerized services and need self-healing, rolling updates, horizontal scaling, and service discovery as primitives.
- You want a declarative, GitOps-friendly deployment model with a large ecosystem (Helm, Argo CD, Prometheus, cert-manager).
- You need portability across clouds and on-prem behind one API.
- You're building a platform for other teams and want CRDs/operators as the extension mechanism.

**Avoid when:**
- You run a handful of services — a PaaS (Fly.io, Render, Cloud Run, App Runner) or plain VMs/Nomad will cost far less operational overhead.
- You have no platform/SRE capacity; Kubernetes shifts complexity onto operators, it does not remove it.
- Your workload is a single monolith or a static site — the orchestration machinery is pure overhead.
- You need hard real-time or specialized scheduling that fights the default scheduler more than it helps.

## Alternatives

- hashicorp/nomad — simpler single-binary orchestrator; use it when you want scheduling without the full Kubernetes API surface and can forgo the CRD ecosystem.
- docker/compose — single-host multi-container definition; use it for local dev and small deployments that never need a cluster.
- **Amazon ECS** — AWS-native orchestration; use it when you're all-in on AWS and want less to operate than self-managed K8s.
- **HashiCorp/Docker Swarm** — minimal clustering built into Docker; use it for small clusters where Kubernetes is overkill (note: largely in maintenance mode).
- k3s-io/k3s — a certified, lightweight Kubernetes distribution; use it at the edge/IoT or for small clusters where you still want the real K8s API.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015-07 | First stable release; project donated to the CNCF[^1]. |
| 1.6 | 2017-03 | etcd v3, RBAC to beta; scaling to 5,000 nodes. |
| 1.16 | 2019-09 | CRD reaches GA; deprecated-API removals begin in earnest. |
| 1.20 | 2020-12 | Dockershim deprecation announced. |
| 1.24 | 2022-05 | Dockershim removed; CRI runtimes (containerd/CRI-O) required[^4]. |
| 1.25 | 2022-08 | PodSecurityPolicy removed, replaced by Pod Security Admission. |
| 1.27 | 2023-04 | SeccompDefault, and the start of the ~14-month support window norm. |
| 1.30 | 2024-04 | Continued GA of storage/scheduling features. |
| 1.33 | 2025-04 | Recent minor in the quarterly cadence. |

## References

[^1]: Google, "Google Open Sources Its Secret Weapon in Cloud Computing" / Borg heritage. Brendan Burns et al.; see also the Borg paper. https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/
[^2]: Kubernetes documentation, "Kubernetes Components". https://kubernetes.io/docs/concepts/overview/components/
[^3]: kubernetes/kubernetes README — "Use of the `k8s.io/kubernetes` module … as libraries is not supported." https://github.com/kubernetes/kubernetes/blob/master/README.md
[^4]: Kubernetes blog, "Dockershim Removal is Coming" / "Updated: Dockershim Removal FAQ". https://kubernetes.io/blog/2022/02/17/dockershim-faq/

## Tags

container-orchestration, kubernetes, go, cncf, distributed-systems, cloud-native, devops, infrastructure, scheduler, declarative-api, containers
