# k3s-io/k3s

> A CNCF-hosted Kubernetes distribution packaged as a single <100 MB binary, aimed at edge, IoT, CI, and resource-constrained clusters.

[GitHub repo](https://github.com/k3s-io/k3s) ·
[Official website](https://k3s.io) ·
[License: Apache-2.0](https://github.com/k3s-io/k3s/blob/main/LICENSE)

## Overview

K3s is a fully conformant Kubernetes distribution, not a fork — it tracks upstream Kubernetes closely and carries only a small patch set (the project states well under 1000 lines) plus opinionated defaults[^1]. It was created at Rancher Labs (Darren Shepherd) and announced in early 2019, then donated to the CNCF as a sandbox project in 2020[^2]. Rancher was acquired by SUSE in 2020, and SUSE remains the primary corporate sponsor. With ~33k stars it is the most-adopted lightweight Kubernetes distribution.

The defining idea is packaging: the entire non-containerized control plane and node agent — API server, controller-manager, scheduler, kubelet, kube-proxy, containerd, flannel, CoreDNS — run inside a single `k3s` process rather than as separate binaries and systemd units. This is what buys the reduced memory footprint and single-file install. It also ships SQLite as the default datastore via a shim (Kine) instead of requiring etcd, which is what makes a one-command single-node cluster viable[^1].

The tradeoff is coupling. Because components are compiled into one release artifact, you cannot independently upgrade containerd, etcd, or the Kubernetes version — they move together at k3s's release cadence. K3s is opinionated (Traefik ingress, ServiceLB/klipper-lb, local-path storage, kube-router for NetworkPolicy) and those choices are enabled by default, so "lightweight" in practice means "curated," not "minimal."

## Getting Started

```bash
# Single-node server. kubeconfig lands at /etc/rancher/k3s/k3s.yaml
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
```

Join a worker (agent) node, using the token from the server:

```bash
# On the server: cat /var/lib/rancher/k3s/server/node-token
curl -sfL https://get.k3s.io | \
  K3S_URL=https://myserver:6443 K3S_TOKEN=<node-token> sh -
```

For a highly available control plane, the first server must initialize the embedded etcd cluster (use an odd number of servers, minimum 3):

```bash
curl -sfL https://get.k3s.io | sh -s - server --cluster-init
```

## Architecture / How It Works

K3s is a supervisor around upstream Kubernetes. The `k3s server` process launches the API server, scheduler, controller-manager, and (unless disabled) the datastore, then runs the kubelet and containerd in-process. `k3s agent` runs only the node-side components. Worker↔server communication is tunneled over a websocket, so the kubelet API port does not need to be exposed on workers[^1].

Key internals:

- **Kine** — an etcd-shim (`etcd`-compatible gRPC API backed by SQLite, MariaDB/MySQL, or PostgreSQL). This is how K3s replaces etcd with a relational datastore. Kine translates the etcd watch/revision model onto SQL, which works well but adds a translation layer and is not a drop-in performance match for real etcd under heavy write load[^3].
- **Embedded etcd** — for HA without an external database, K3s can run a managed etcd cluster across the server nodes. This replaced the earlier experiment with dqlite, which was dropped. Etcd (not SQLite) is required for multi-server HA using the embedded datastore.
- **Bundled containerd** — K3s ships and manages its own containerd; it does not use a system Docker/containerd by default. Images and state live under `/var/lib/rancher/k3s`.
- **Auto-deploying manifests** — anything dropped in `/var/lib/rancher/k3s/server/manifests` is applied and reconciled automatically. Bundled add-ons (Traefik, CoreDNS, metrics-server, local-path, ServiceLB) are delivered this way, Traefik via a `HelmChart` CRD reconciled by the embedded helm-controller.
- **Networking** — flannel (VXLAN backend) is the default CNI; kube-router provides NetworkPolicy enforcement; klipper-lb ("ServiceLB") gives `type: LoadBalancer` support by binding host ports on nodes.

The single-process model is the coupling story: the version of every bundled component is pinned to the k3s release you install.

## Production Notes

**SQLite is single-server only.** The default datastore does not do HA. Moving to multiple control-plane nodes means either `--cluster-init` (embedded etcd) or pointing every server at a shared external DB (`--datastore-endpoint`). You cannot casually migrate a running SQLite cluster to etcd in place — plan the datastore up front.

**Etcd hates slow disks.** The most common edge footgun: running embedded etcd on SD cards or cheap USB storage (Raspberry Pi, low-end nodes) produces high fsync latency, which manifests as etcd leader elections, `apply request took too long` warnings, and cluster instability. Etcd wants low-latency local SSD/NVMe. For genuinely small edge nodes, prefer a single SQLite server over a 3-node etcd HA setup on flash media.

**Editing bundled add-ons.** Because Traefik/CoreDNS/etc. are auto-reconciled from the manifests directory, hand-editing the deployed resources gets reverted on restart. Customize via a `HelmChartConfig`, or disable the add-on (`--disable=traefik`, `--disable=servicelb`) and manage your own. To stop a bundled manifest from being reapplied, remove it and add a `.skip` file.

**Traefik v1 → v2 upgrade break.** Older K3s shipped Traefik v1; the move to Traefik v2 changed the ingress/CRD model and broke existing IngressRoute and annotation configs for clusters that upgraded across that boundary[^4]. Pin or manage Traefik yourself if ingress stability matters across upgrades.

**cgroups and kernel prerequisites.** On some distros you must enable cgroup memory/cpuset at boot (e.g. editing `cmdline.txt` on Raspberry Pi OS) or the kubelet fails to start. K3s needs a sane kernel with cgroups mounted, not a full container platform.

**Upgrades.** Re-running the install script with a new `INSTALL_K3S_VERSION`, or the `system-upgrade-controller` with Plan CRDs, are the two supported paths. Because Kubernetes and all add-ons move together, read the release notes for the specific `+k3s<n>` build — component bumps ride along with the k8s version.

**Air-gapped installs** require pre-staging the airgap images tarball and the binary; the convenience `get.k3s.io` script assumes internet egress.

## When to Use / When Not

**Use when:**
- Edge, IoT, CI runners, homelab, or dev clusters where a full kubeadm/etcd stack is overkill.
- You want a single-binary, single-command Kubernetes with sane batteries-included defaults.
- ARM / small-node fleets (arm64, armhf, and s390x binaries are published).
- You need conformant Kubernetes but not a specific upstream component topology.

**Avoid when:**
- You need to independently version or swap the control-plane components — the single artifact pins them.
- You require a CIS-hardened / government-grade distribution out of the box — see rke2 (the sibling project) instead.
- You are running a large, write-heavy cluster where full etcd tuning and separation of concerns matter more than a small footprint.
- Your platform team has standardized on a managed cloud Kubernetes (EKS/GKE/AKS) — K3s solves a different problem.

## Alternatives

- k0sproject/k0s — similar single-binary lightweight distro; no bundled ingress/LB by default, so a cleaner base if you want fewer opinions.
- rancher/rke2 — same team's security-hardened distro (CIS/FIPS, etcd-only, no single-process packing); use when compliance beats footprint.
- canonical/microk8s — snap-based lightweight k8s; use on Ubuntu/snap-centric fleets.
- kubernetes-sigs/kind — Kubernetes in Docker; use for ephemeral CI and local testing, not production.
- siderolabs/talos — immutable API-managed OS + Kubernetes; use when you want the whole node to be appliance-like.

## History

| Version | Date | Notes |
|---------|------|-------|
| Announce | 2019-02 | K3s introduced by Rancher Labs (Darren Shepherd)[^2]. |
| v1.0.0 | 2019-11 | First stable release. |
| CNCF sandbox | 2020 | Donated to the CNCF as a sandbox project[^2]. |
| Traefik v2 | ~2021 | Default ingress moved v1 → v2; breaking for upgraders[^4]. |
| Embedded etcd | 2020–2021 | Managed etcd for HA; dqlite experiment dropped. |
| Ongoing | 2026 | Tracks upstream k8s; patches within a week, minors within ~30 days[^1]. |

## References

[^1]: K3s README — "What is this?", release cadence, single-binary design. https://github.com/k3s-io/k3s/blob/main/README.md
[^2]: CNCF, "K3s" project profile / sandbox acceptance. https://www.cncf.io/projects/k3s/
[^3]: Kine — etcd-to-SQL shim used by K3s. https://github.com/k3s-io/kine
[^4]: K3s documentation — networking / Traefik ingress add-on. https://docs.k3s.io/networking/networking-services

## Tags

kubernetes, k8s, go, container-orchestration, edge, iot, lightweight, cncf, distribution, containerd, self-hosted, devops
