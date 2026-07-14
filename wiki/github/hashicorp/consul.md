# hashicorp/consul

> Service discovery, health checking, a KV store, and an Envoy-based service mesh in a single Go binary — with a controversial 2023 license change hanging over it.

[GitHub repo](https://github.com/hashicorp/consul) ·
[Official website](https://www.consul.io) ·
[License: BUSL-1.1](https://github.com/hashicorp/consul/blob/main/LICENSE)

## Overview

Consul is a distributed control plane for connecting, configuring, and securing services across dynamic infrastructure[^1]. First released by HashiCorp in 2014, it started as a service-discovery and health-checking system with a built-in key/value store, and grew into a full service mesh ("Consul Connect", later just the mesh) built on Envoy sidecar proxies. It runs as a single Go binary on Linux, macOS, FreeBSD, Solaris, and Windows, and deploys on VMs, bare metal, Kubernetes, Nomad, and ECS.

The defining tension is **consistency vs. availability at scale**. Consul's server cluster uses the Raft consensus protocol for a strongly consistent catalog and KV store, while node/service membership and failure detection use the Serf gossip protocol (an eventually-consistent, epidemic broadcast). This split lets Consul stay available for reads and gossip even during a Raft leader election, but it also means operators must reason about two very different consistency models in one system. Sizing the Raft quorum, keeping gossip healthy across thousands of nodes, and understanding which API calls hit the strongly-consistent path are the core operational skills.

The second defining fact about Consul in 2026 is **its license**. In August 2023 HashiCorp relicensed Consul (and Terraform, Vault, Nomad, and others) from the open-source MPL-2.0 to the Business Source License 1.1 (BUSL-1.1), a source-available license that forbids competing commercial/hosted use[^2]. Everything from Consul 1.17 onward is BUSL; 1.16.x was the last MPL-2.0 release. Unlike Terraform (forked as OpenTofu), Consul did not gain a widely adopted community fork. HashiCorp was acquired by IBM in a deal that closed in early 2025[^3], which did not reverse the license change.

## Getting Started

```bash
# macOS
brew tap hashicorp/tap && brew install hashicorp/tap/consul
# or download a static binary from https://developer.hashicorp.com/consul/install
```

Run a single-node dev agent (in-memory, never for production):

```bash
consul agent -dev
```

Register a service and query it over DNS or HTTP:

```bash
# register via a config file, or the HTTP API:
curl -X PUT http://127.0.0.1:8500/v1/agent/service/register \
  -d '{"Name":"web","Port":8080,"Check":{"HTTP":"http://localhost:8080/health","Interval":"10s"}}'

# discover healthy instances over DNS (SRV/A records on port 8600):
dig @127.0.0.1 -p 8600 web.service.consul

# or over HTTP — only passing health checks are returned:
curl http://127.0.0.1:8500/v1/health/service/web?passing
```

## Architecture / How It Works

Consul runs an **agent** on every participating node, in one of two modes:

- **Server agents** — form the Raft cluster. They hold the authoritative catalog, KV store, ACLs, and mesh config. Run 3 or 5 per datacenter (odd numbers for quorum; 3 tolerates one failure, 5 tolerates two).
- **Client agents** — lightweight, stateless forwarders that run alongside application workloads, perform local health checks, and relay RPCs to servers. On Kubernetes this role is increasingly replaced by **Consul Dataplane** (introduced 1.14, 2022), which removes the per-node client agent and injects Envoy directly, talking to servers over gRPC[^4].

Two consensus layers coexist:

1. **Raft** (servers only) — strongly consistent log for catalog writes, KV, ACL tokens, and mesh intentions. All writes route to the leader. A partition that loses quorum blocks writes until it heals.
2. **Serf / gossip** (all agents) — a SWIM-derived gossip protocol for membership, failure detection, and event broadcast. There is a LAN gossip pool per datacenter and a WAN gossip pool across datacenters for federation.

**Service mesh.** Connect issues short-lived mTLS certificates from a built-in or Vault-backed CA, and programs an Envoy sidecar per service. Authorization is expressed as **intentions** (allow/deny between service identities). Layer-7 traffic management (routing, splitting, retries) is configured through `service-router`, `service-splitter`, and `service-resolver` config entries. **Transparent proxy** mode (on Kubernetes) redirects pod traffic through Envoy via iptables so applications need no code changes.

**Multi-datacenter.** WAN gossip federation joins datacenters into a mesh; mesh gateways tunnel sidecar traffic between them without exposing every service publicly. This is Consul's historical differentiator over Kubernetes-only meshes.

The DNS interface (port 8600), HTTP/API (8500), and gRPC surface all read from the same catalog. Most reads default to a slightly stale "leader-forwarded" consistency mode for throughput; you opt into linearizable reads with `?consistent`.

## Production Notes

**Raft quorum is the availability floor.** Losing a majority of servers means no writes and, eventually, a read-only cluster. Run servers across failure domains, monitor `consul.raft.leader.lastContact`, and take snapshots (`consul snapshot save`) on a schedule — a lost quorum without a snapshot means catalog reconstruction from agents, which is painful.

**Gossip does not love huge flat clusters.** Tens of thousands of agents in one gossip pool increase CPU and network churn and make flapping (rapid failed/alive transitions) more likely under network stress. Tune `gossip` timings for high-latency links, and prefer network segments or multiple datacenters over one giant pool.

**Server memory scales with the catalog.** Servers keep the full state (services, nodes, KV, ACLs) in memory and in the Raft log. Large catalogs, high KV churn, or many mesh config entries drive RAM and snapshot size. Watch heap and the Raft log size; frequent large writes force frequent snapshots and log truncation.

**ACLs default-allow historically bit people.** Early Consul shipped with ACLs off; unauthenticated agents could read/write the catalog and KV. Modern deployments must bootstrap ACLs with a default-deny policy and distribute agent tokens — retrofitting ACLs onto a running cluster is disruptive.

**Upgrades are ordered and version-sensitive.** Autopilot manages server introduction, but you upgrade servers one at a time, respecting the Raft protocol version and never skipping too many minor versions. Envoy has a supported-version matrix per Consul release; upgrading Consul can force a coordinated Envoy/dataplane upgrade.

**Licensing affects your dependency policy, not just legality.** BUSL-1.1 is source-available, not OSI open-source. Self-hosting and internal use are permitted, but offering Consul as a competing hosted service is not, and some organizations' OSS-compliance rules now flag it. If that matters to you, pin to 1.16.x (last MPL) or evaluate alternatives — there is no mature community fork to fall back to.

## When to Use / When Not

**Use when:**
- You need service discovery and health checking across mixed VM + Kubernetes + bare-metal estates, not just inside one Kubernetes cluster.
- You need multi-datacenter federation and cross-DC service mesh with mesh gateways.
- You want discovery, KV config, and mesh from one control plane rather than assembling separate tools.
- You already run the HashiCorp stack (Nomad, Vault) and want tight integration.

**Avoid when:**
- You are Kubernetes-only and want the most idiomatic mesh — Istio, Linkerd, or Cilium fit the K8s model more naturally.
- You only need a coordination/KV store — etcd or ZooKeeper are lighter and purpose-built.
- The BUSL-1.1 license is disqualifying under your open-source policy.
- You cannot staff the operational burden of a Raft quorum plus a gossip layer plus Envoy.

## Alternatives

- istio/istio — use instead when you are Kubernetes-native and want the most feature-rich, widely adopted mesh (at higher complexity and resource cost).
- linkerd/linkerd2 — use instead when you want a lighter, simpler Kubernetes mesh with a purpose-built Rust micro-proxy instead of Envoy.
- etcd-io/etcd — use instead when you only need a strongly consistent KV/coordination store (it is Kubernetes' own backing store) and no mesh or discovery layer.
- cilium/cilium — use instead when you want eBPF-based networking, observability, and a sidecar-less mesh on Kubernetes.
- apache/zookeeper — use instead when you need a battle-tested coordination primitive for legacy JVM systems (Kafka, Hadoop) rather than service discovery.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2014-04 | Initial release: service discovery, health checks, KV, Raft + Serf[^1]. |
| 1.0 | 2017-11 | First stable major release. |
| 1.2 | 2018-06 | Connect service mesh introduced (mTLS + intentions). |
| 1.6 | 2019-09 | Mesh gateways for cross-datacenter mesh traffic. |
| 1.9 | 2020-11 | Layer-7 traffic management, ingress/terminating gateways maturing. |
| 1.14 | 2022-11 | Consul Dataplane — sidecar without a per-node client agent[^4]. |
| 1.16 | 2023-06 | Last release under MPL-2.0 open-source license. |
| 1.17 | 2023-10 | First release under BUSL-1.1[^2]. |

## References

[^1]: HashiCorp, "Consul Announcement" — 2014-04. https://www.hashicorp.com/blog/consul-announcement
[^2]: HashiCorp, "HashiCorp adopts Business Source License" — 2023-08-10. https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license
[^3]: IBM, "IBM completes acquisition of HashiCorp" — 2025-02-27. https://newsroom.ibm.com/2025-02-27-ibm-completes-acquisition-of-hashicorp
[^4]: HashiCorp, "Consul Dataplane" documentation. https://developer.hashicorp.com/consul/docs/connect/dataplane

## Tags

go, service-discovery, service-mesh, distributed-systems, raft, gossip, envoy, key-value-store, kubernetes, api-gateway, busl, hashicorp
