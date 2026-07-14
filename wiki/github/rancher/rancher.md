# rancher/rancher

> A Kubernetes management platform: one control plane that provisions, imports, and governs many downstream clusters.

[GitHub repo](https://github.com/rancher/rancher) ·
[Official website](https://rancher.com) ·
[License: Apache-2.0](https://github.com/rancher/rancher/blob/main/LICENSE)

## Overview

Rancher is a management layer that sits *above* Kubernetes clusters rather than being a cluster itself. Its job is fleet-scale operations: provision new clusters on public clouds, vSphere, or bare metal; import existing ones; and then present a single pane for authentication, RBAC, monitoring, and app deployment across all of them. It is written in Go and has been under active development since 2014, making it one of the oldest surviving projects in the container-orchestration space. It is now owned and maintained by SUSE, which acquired Rancher Labs in 2020[^1].

The project has lived two distinct lives. Rancher 1.x (2014–2018) was a Docker-centric orchestrator with its own scheduler called *Cattle*, competing with Docker Swarm and Mesos. Rancher 2.0 (2018) threw that out and rebuilt the product entirely around Kubernetes: Cattle the scheduler was retired, and Rancher became a Kubernetes-of-Kubernetes control plane[^2]. The `cattle` topic tag and many internal names are fossils from that first life. Anyone reading old blog posts should check whether they predate the 2.0 rewrite, because almost nothing carries over.

The defining tension is centralization. Rancher gives you unified auth and a friendly UI over dozens of clusters, but in doing so it inserts itself as an authentication proxy in front of every cluster's API server. That convenience is also a coupling and a blast radius, and it is the source of most of Rancher's operational war stories.

## Getting Started

The single-container Docker install is the fastest way to see the UI. It is explicitly for evaluation, not production:

```bash
sudo docker run -d --restart=unless-stopped \
  -p 80:80 -p 443:443 --privileged \
  rancher/rancher:v2.14.3
# then open https://localhost and set the admin password
```

Production installs run Rancher as a Helm chart on a dedicated Kubernetes cluster:

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm install rancher rancher-stable/rancher \
  --namespace cattle-system --create-namespace \
  --set hostname=rancher.example.com \
  --set replicas=3
```

Once running, you register downstream clusters either by having Rancher provision them (node/cluster drivers) or by running a one-line `kubectl apply` of the cluster agent manifest against an existing cluster to import it.

## Architecture / How It Works

Rancher is itself a set of Kubernetes controllers. It runs on a *local* (management) cluster and manages *downstream* clusters through two agents deployed into each: `cattle-cluster-agent` (talks to the cluster's API server) and `cattle-node-agent` (per-node tasks such as provisioning and backups). Communication is a persistent tunnel initiated *from* the downstream side, so downstream API servers do not need to be publicly reachable.

Several homegrown frameworks underpin the codebase, and their names show up constantly in issues:

- **Wrangler** — the controller/code-generation framework Rancher uses across all its projects; the equivalent of writing raw client-go controllers by hand.
- **Norman** — the original Rancher 2.x API framework (schemas, generic CRUD). Being superseded but still present.
- **Steve** — the newer aggregation API that backs the current dashboard, proxying and caching downstream cluster resources.
- **Projects** — a Rancher-only abstraction that groups namespaces for RBAC. It has no Kubernetes-native equivalent, so project-scoped permissions exist only while Rancher is in the path.

Cluster provisioning has itself been rewritten. **Provisioning v1** drove RKE1 (Rancher Kubernetes Engine, a Docker-based distro). **Provisioning v2** is built on upstream Cluster API (CAPI) and provisions RKE2 and K3s clusters[^3]. RKE2 is the security-hardened, CNCF-conformant successor; K3s is the lightweight single-binary distribution aimed at edge and CI. Both K3s and RKE2 are separate SUSE/Rancher projects, not code in this repo.

This repository is a **meta-repo**: it is primarily used for packaging and pulls the bulk of the actual logic from dozens of sibling modules listed in `go.mod`. Reading a single file here rarely tells the whole story.

## Production Notes

**Run Rancher on its own cluster.** The strongly recommended topology is a dedicated three-node management cluster whose only job is Rancher. Installing Rancher onto a cluster that also runs your workloads couples control-plane availability to workload load and makes upgrades far riskier.

**The auth proxy is a single point of failure for `kubectl`.** Kubeconfigs handed out by Rancher point at Rancher's proxy, not directly at the downstream API server. If the management cluster is down, those kubeconfigs stop working even though the downstream cluster itself is healthy. Keep a break-glass direct kubeconfig for every production cluster.

**Certificate and version skew are the classic outages.** Rancher pins tight compatibility windows between the Rancher version, the Kubernetes version of downstream clusters, and cert-manager. Internal CA/agent certificates have finite lifetimes; expiry or a cert-manager mismatch during install has repeatedly caused agents to disconnect. Always cross-check the published support matrix before any upgrade[^4].

**Upgrades are sequential and unforgiving.** You generally cannot skip minor versions (e.g. 2.7 → 2.9); you step through them. Each Rancher minor also constrains which downstream Kubernetes versions are supported, so a Rancher upgrade often forces coordinated cluster upgrades. Take etcd snapshots first.

**Scale limits are real.** A single Rancher install manages a bounded number of clusters and nodes before the management cluster's etcd and the Steve cache become the bottleneck. Large fleets shard across multiple Rancher installs or lean on Fleet (GitOps) to reduce per-cluster API churn.

**RKE1 is end-of-life.** New clusters should be RKE2 or K3s; RKE1/provisioning-v1 is deprecated and being phased out, so greenfield RKE1 is a dead end.

## When to Use / When Not

**Use when:**
- You operate many Kubernetes clusters across mixed infrastructure (cloud + vSphere + bare metal) and want one auth/RBAC/UI plane over all of them.
- You want turnkey cluster provisioning with RKE2/K3s plus batteries-included monitoring, logging, and GitOps (Fleet).
- Non-expert teams need a UI to operate clusters without living in `kubectl`.

**Avoid when:**
- You run a single cluster: the management overhead and extra failure mode are not worth it — use plain `kubectl`, Lens, or a lightweight dashboard.
- You are fully committed to one cloud's managed Kubernetes (EKS/GKE/AKS) and its native tooling; Rancher's value is cross-cluster and cross-cloud.
- You want a minimal, few-moving-parts stack: Rancher is a large system with its own controllers, agents, and abstractions to learn and keep patched.

## Alternatives

- k3s-io/k3s — a Rancher/SUSE project too, but a Kubernetes *distribution*, not a manager; reach for it when you want lightweight clusters, not a fleet control plane.
- portainer/portainer — use instead when you want a simpler container/Kubernetes UI without cluster provisioning or multi-cluster governance.
- kubesphere/kubesphere — a comparable full-featured Kubernetes platform with a dashboard; consider when you want an all-in-one distro rather than a manager over external clusters.
- argoproj/argo-cd — use for GitOps-only needs; it overlaps Rancher's Fleet component without the rest of the platform.
- red-hat / OpenShift (OKD) — use when you want a single opinionated enterprise distribution with integrated build/CI, rather than a neutral manager over heterogeneous clusters.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016 | Docker-era orchestrator with the Cattle scheduler[^2]. |
| 2.0 | 2018-05 | Full rewrite around Kubernetes; Cattle scheduler retired[^2]. |
| — | 2020-07 | Rancher Labs acquired by SUSE[^1]. |
| 2.5 | 2020-10 | Rancher shipped as a Helm app; new Cluster Explorer dashboard, monitoring v2. |
| 2.6 | 2021-10 | Redesigned UI, extensions, provisioning-v2 (CAPI + RKE2/K3s) groundwork[^3]. |
| 2.7 | 2022-10 | Provisioning v2 maturity; RKE2 as the recommended distro. |
| 2.14.3 | 2026 | Current stable line at time of writing[^5]. |

## References

[^1]: SUSE, "SUSE Completes Acquisition of Rancher Labs" — 2020-12-01. https://www.suse.com/news/SUSE-Completes-Rancher-Acquisition/
[^2]: Rancher Labs, "Rancher 2.0" announcement and Kubernetes pivot. https://www.rancher.com/blog/2018/2018-05-01-rancher-2-0-ga
[^3]: Rancher docs, provisioning clusters with RKE2/K3s (Cluster API). https://ranchermanager.docs.rancher.com/pages-for-subheaders/provisioning-rke2-clusters
[^4]: Rancher Support Matrix (version compatibility across Rancher, Kubernetes, and OS). https://www.suse.com/suse-rancher/support-matrix/
[^5]: Rancher releases — stable tag reported by the repository README. https://github.com/rancher/rancher/releases

## Tags

go, kubernetes, container-orchestration, multi-cluster, devops, cluster-management, platform, rke2, k3s, gitops, suse
