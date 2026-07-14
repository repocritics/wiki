# cilium/cilium

> eBPF-based networking, security, and observability for Kubernetes — the dataplane that replaces kube-proxy and iptables with in-kernel programs.

[GitHub repo](https://github.com/cilium/cilium) ·
[Official website](https://cilium.io) ·
[License: Apache-2.0](https://github.com/cilium/cilium/blob/main/LICENSE)

## Overview

Cilium is a CNI (Container Network Interface) plugin and dataplane for Kubernetes, built on eBPF[^1]. Instead of programming the Linux network stack through iptables rules and kube-proxy — the traditional approach, which scales poorly as service and pod counts grow — Cilium loads eBPF bytecode directly into the kernel at hook points (network I/O, sockets, tracepoints) and makes routing, load-balancing, and policy decisions there. It was started by Thomas Graf and the team that became Isovalent in 2015, reached 1.0 in 2018, graduated CNCF in 2023 (the first CNI to do so)[^2], and Isovalent was acquired by Cisco the same year[^3].

The defining architectural choice is **identity-based security decoupled from IP addressing**. Cilium assigns a numeric security identity to each set of endpoints sharing the same labels, and enforces policy on identities rather than IPs. This means a policy does not need to be rewritten every time a pod is rescheduled to a new address, and enforcement scales with the number of distinct identities rather than the number of pods. Network policy spans L3 through L7 — the same engine can filter on labels, ports, DNS names (FQDN policy), and HTTP method/path or gRPC calls.

The tradeoff is depth of coupling to the kernel. Cilium's feature set is gated by Linux kernel version: XDP acceleration, BPF host routing, and various kube-proxy-replacement paths each require specific kernel capabilities, and debugging a misbehaving dataplane means reasoning about eBPF maps and program attachment rather than reading `iptables -L`. Cilium is powerful precisely because it lives in the kernel, and operationally demanding for the same reason.

## Getting Started

Install with the Cilium CLI against an existing cluster (kernel and mount prerequisites apply):

```bash
# Install the cilium CLI (macOS example)
brew install cilium-cli

# Install Cilium into the current kube context
cilium install --version 1.19.5

# Verify dataplane health and run connectivity tests
cilium status --wait
cilium connectivity test
```

A minimal L7-aware network policy — allow pods labeled `app=frontend` to `GET /api` on `app=backend`, deny everything else:

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: frontend-to-backend
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              - method: "GET"
                path: "/api"
```

## Architecture / How It Works

Cilium runs as a **DaemonSet** (`cilium-agent` on every node) plus a cluster-wide **operator**. The agent compiles and loads eBPF programs, manages endpoints, and syncs state; the operator handles cluster-scoped tasks like IPAM allocation and CRD garbage collection. State that must be shared across nodes (identities, service backends) is stored either in a **kvstore** (etcd) or, increasingly, in Kubernetes CRDs directly (CRD-backed mode, the common default).

Core subsystems:

- **Datapath modes.** Two ways to move pod traffic between nodes: *overlay* (encapsulation via VXLAN or Geneve — works on almost any underlay, only needs IP connectivity between hosts) and *native routing* (the host's own routing table, requires the network to route pod CIDRs, often paired with BGP advertisement).
- **kube-proxy replacement.** Service ClusterIP/NodePort/LoadBalancer handling is implemented in eBPF hash tables. East-west traffic is load-balanced at the socket layer (`connect()` rewrite) so there is no per-packet NAT; north-south paths support XDP, Direct Server Return (DSR), and Maglev consistent hashing.
- **Identity and policy.** Labels → security identity → policy verdict. L7 rules are enforced by transparently redirecting matched traffic to a per-node Envoy proxy; L3/L4 rules stay entirely in eBPF.
- **Hubble.** The observability layer, reading flow events from the eBPF datapath. Provides a flow log, service map, and Prometheus metrics with identity/label context rather than bare IPs. Runs as an optional component with its own CLI and UI.
- **ClusterMesh.** Connects multiple clusters into a shared identity and service-discovery domain for failover and global services.
- **Encryption.** Transparent node-to-node encryption via IPsec or WireGuard; mutual authentication between workloads.

The kernel dependency is load-bearing throughout: which of these features are available, and how fast they run, is a function of the running kernel, not just the Cilium version.

## Production Notes

**Kernel version dictates the feature matrix.** Cilium runs on older kernels but many high-value features (full kube-proxy replacement, BPF host routing, bandwidth manager, some XDP paths) require newer ones. The docs maintain a per-feature kernel requirement table; treating "Cilium 1.x is installed" as equivalent to "all 1.x features work" is a recurring source of surprise. Managed Kubernetes nodes pin you to the provider's kernel.

**kube-proxy replacement has partial and strict modes.** Running Cilium alongside kube-proxy vs. fully replacing it are different operational states. Full replacement needs `k8sServiceHost`/`k8sServicePort` set correctly (the API server address without kube-proxy to reach it) and specific kernel support; getting this half-configured produces services that resolve but do not balance as expected.

**eBPF map sizing and conntrack.** The connection-tracking and policy maps have fixed sizes tuned by config. At high connection churn or scale, default map limits can be exceeded; sizing these is part of large-cluster tuning, and changes can require agent restarts that briefly disrupt the datapath.

**Upgrades deserve the upgrade guide.** Minor-version upgrades can involve datapath and CRD schema changes. Cilium maintains only the last three minor releases; older minors go EOL[^1]. Pre-pulling images, reading the per-version upgrade notes, and testing in a non-prod cluster are not optional at scale — a botched datapath upgrade affects all pod networking on the node.

**Debugging is different.** There is no `iptables -L` to read. The equivalents are `cilium monitor` (live datapath events), `cilium status`, `hubble observe` (flow logs with drop reasons), and endpoint/identity inspection. Teams new to eBPF should budget time to learn these before an incident, not during one.

**Envoy for L7.** L7 policy and Gateway API/Ingress route traffic through an embedded Envoy proxy per node. This adds CPU and memory cost and a second moving part; L3/L4-only deployments avoid it entirely.

## When to Use / When Not

**Use when:**
- You run Kubernetes at a scale where iptables/kube-proxy is a measured bottleneck.
- You need identity-based, L3–L7 network policy including DNS/FQDN and HTTP-aware rules.
- You want deep flow observability (Hubble) tied to workload identity, not IPs.
- You need multi-cluster connectivity (ClusterMesh) or transparent encryption as first-class features.

**Avoid when:**
- You need a minimal CNI for a small cluster and don't want to operate an eBPF dataplane — the operational surface is real.
- Your nodes are pinned to old kernels that gate the features you actually want.
- Your team has no appetite for learning eBPF-based troubleshooting and there is no in-house networking depth.
- You only need basic pod connectivity with no policy — simpler plugins do that with far less to run.

## Alternatives

- projectcalico/calico — the other dominant policy-capable CNI; offers both iptables and eBPF dataplanes. Use when you want a mature policy engine with a gentler operational model, or need Calico's routing/BGP maturity without committing fully to eBPF.
- flannel-io/flannel — minimal overlay CNI with no network policy. Use when you want the simplest possible pod networking and will layer policy elsewhere (or not at all).
- antrea-io/antrea — Open vSwitch–based CNI with good Windows-node support. Use when you need OVS integration or first-class Windows networking.
- istio/istio — sidecar service mesh. Use when you need the full mesh feature set (rich traffic management, mTLS everywhere) and accept sidecar overhead, rather than Cilium's proxy-light mesh.
- cilium/tetragon — eBPF runtime security and observability from the same project; complementary rather than a substitute (process/syscall enforcement vs. network dataplane).

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018-04 | First stable release; eBPF dataplane, L3–L7 policy[^1]. |
| 1.6 | 2019-11 | kube-proxy replacement, identity/policy scaling work. |
| 1.9 | 2020-11 | Maglev, egress gateway, host firewall maturing. |
| 1.11 | 2021-12 | Hubble improvements, OpenTelemetry, Cluster Mesh scaling. |
| 1.12 | 2022-07 | Service Mesh, Gateway API, Ingress, mutual authentication preview. |
| 1.14 | 2023-08 | Gateway API GA, mutual auth, BGPv2 direction. |
| — | 2023-10 | Graduated CNCF — first CNI to graduate[^2]. |
| — | 2023-12 | Cisco announces acquisition of Isovalent[^3]. |
| 1.17 | 2025 | Continued datapath/BGP/observability work. |
| 1.18 | 2025 | Maintained stable branch (1.18.11 as of 2026-06)[^1]. |
| 1.19 | 2026 | Latest stable minor (1.19.5); 1.20 in pre-release[^1]. |

Actively developed: ~24.7k stars, ~3.9k forks, and commits landing daily on `main` as of mid-2026[^4]. The repository is one of the busiest in the CNCF graduated tier, with a large maintainer group and heavy issue/PR throughput.

## References

[^1]: Cilium README and stable-release policy (last three minor releases maintained). https://github.com/cilium/cilium
[^2]: CNCF, "Cilium becomes the first CNI to graduate" — 2023-10-11. https://www.cncf.io/announcements/2023/10/11/cloud-native-computing-foundation-announces-cilium-graduation/
[^3]: Cisco, intent to acquire Isovalent — announced 2023-12-21. https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2023/m12/cisco-announces-intent-to-acquire-isovalent.html
[^4]: GitHub API repository metadata for cilium/cilium, fetched 2026-07-14. https://api.github.com/repos/cilium/cilium

## Tags

go, ebpf, kubernetes, cni, networking, network-policy, observability, load-balancing, service-mesh, cncf, security, container-networking
