# hashicorp/nomad

> A single-binary workload orchestrator for containers, standalone binaries, and VMs — the simpler, less opinionated alternative to Kubernetes.

[GitHub repo](https://github.com/hashicorp/nomad) ·
[Official website](https://www.nomadproject.io/) ·
[License: BUSL-1.1](https://github.com/hashicorp/nomad/blob/main/LICENSE)

## Overview

Nomad is a cluster scheduler and workload orchestrator from HashiCorp, first released in 2015[^1]. It places work — Docker containers, raw executables, Java apps, QEMU VMs — onto a fleet of machines, keeps it running through failures, and reschedules it when nodes die. Its defining characteristic is scope: where Kubernetes is a platform with a large surface area (networking, storage, ingress, RBAC, CRDs, a controller ecosystem), Nomad is a scheduler and little else. It ships as one Go binary that acts as either a server or a client, stores its own state via Raft, and requires no external database.

The central tradeoff is minimalism versus ecosystem. Nomad is genuinely easy to stand up and operate — a working cluster is a handful of agents and one HCL job file — and it orchestrates non-containerized workloads as first-class citizens, which Kubernetes does not. In exchange you get a far smaller ecosystem: fewer off-the-shelf operators, fewer managed offerings, less community tooling, and a smaller hiring pool. Networking and secrets are not built in the way they are in Kubernetes; Nomad instead leans on its HashiCorp siblings Consul (service discovery, service mesh) and Vault (secrets), which is elegant if you already run that stack and friction if you don't.

In August 2023 HashiCorp relicensed Nomad — along with Terraform, Consul, Vault, and the rest of its portfolio — from MPL-2.0 to the Business Source License 1.1 (BUSL-1.1)[^2]. The BUSL forbids offering Nomad as a competing managed service; ordinary production use is unaffected. GitHub's license detector reports the repo as "NOASSERTION" because it does not recognize the BUSL. IBM's acquisition of HashiCorp closed in early 2025, placing Nomad under IBM stewardship.

## Getting Started

```bash
# macOS via Homebrew; Linux/Windows: download the single binary from releases
brew install hashicorp/tap/nomad

# Single-node dev agent (server + client in one process, in-memory)
sudo nomad agent -dev
```

```hcl
# example.nomad.hcl — run 3 copies of an nginx container
job "web" {
  group "frontend" {
    count = 3

    network {
      port "http" { to = 80 }
    }

    task "nginx" {
      driver = "docker"
      config {
        image = "nginx:1.27"
        ports = ["http"]
      }
      resources {
        cpu    = 200  # MHz
        memory = 128  # MB
      }
    }
  }
}
```

```bash
nomad job run example.nomad.hcl
nomad job status web
```

## Architecture / How It Works

A Nomad deployment is two agent roles running the same binary:

- **Servers** form a Raft peer set (3 or 5 for HA)[^3]. They hold all cluster state — jobs, allocations, node registrations, evaluations — in an in-memory store backed by Raft's log. One server is the elected leader and runs the schedulers.
- **Clients** are the worker nodes. Each fingerprints its host (CPU, memory, drivers, devices), registers with the servers, and runs the allocations assigned to it.

Scheduling is a pipeline of **jobs → evaluations → allocations**. Submitting or changing a job creates an evaluation; a scheduler processes it, computes placements by feasibility-filtering and then ranking nodes (bin-packing by default, with optional `spread` and `affinity` stanzas), and emits allocations that clients pull and execute. The scheduler is **optimistically concurrent**: it computes a plan against a snapshot and submits it to the leader, which rejects the plan if the underlying state changed — this is what lets Nomad schedule at high throughput without a global lock[^4]. There are distinct scheduler types: `service` (long-running), `batch`, `system` (one alloc per eligible node, like a Kubernetes DaemonSet), and `sysbatch`.

**Task drivers** are pluggable and run the actual work: `docker`, `exec`/`exec2` (isolated fork-exec), `raw_exec` (no isolation), `java`, `qemu`, plus out-of-tree drivers like `podman`. **Device plugins** expose GPUs, FPGAs, and TPUs to the scheduler. Storage is host volumes or CSI plugins. Multi-region **federation** uses Serf gossip between server clusters, letting one job target multiple regions.

State replication is Raft within a region; cross-region is gossip, not consensus. Since 1.3, Nomad has a built-in service catalog and (since 1.4) an encrypted key-value store called **Variables**, which removed the hard dependency on Consul and Vault for basic service discovery and secrets[^5]. Workload identity (1.7+) issues each allocation a signed JWT, which is the modern, tokenless way to authenticate to Consul and Vault.

## Production Notes

- **Nomad is a scheduler, not a platform.** Out of the box you get placement and lifecycle management. Load balancing, ingress, DNS, service mesh, and secrets are separate decisions — usually Consul + a load balancer, or native service discovery for simpler setups. Teams migrating from Kubernetes are frequently surprised by how much they must assemble themselves.
- **Server sizing and Raft.** Run 3 or 5 servers, never an even number (split-brain risk). Because state lives in memory and is Raft-replicated, very large clusters push memory and Raft snapshot pressure on the leader. Nomad has been demonstrated at 10,000+ nodes and, in staged challenges, over a million containers[^6], but reaching that requires tuning `raft_multiplier`, GC intervals, and eval batching.
- **Job updates are deliberate.** The `update` stanza controls rolling deploys, canaries, and `auto_revert`. Without a configured stanza, an update can replace all allocations at once. Setting `max_parallel`, `health_check`, and `canary` is close to mandatory for service jobs.
- **Bin-packing can strand capacity.** The default bin-packing ranker maximizes density, which is efficient but can concentrate load; use `spread` for anti-affinity across racks/AZs and set realistic `resources` — Nomad enforces memory limits and CPU is a relative share, not a hard cap unless configured.
- **The HashiCorp coupling cuts both ways.** Consul/Vault integration is excellent if you run them; if you don't, you are maintaining service discovery and secrets some other way. Workload identity (1.7) simplified the token bootstrapping that used to be a common source of production pain.
- **Upgrades.** Follow the documented order (servers before clients, one minor version at a time) and check the upgrade guide for state-format changes. The 1.4 → later transition around Variables and the 1.7 workload-identity migration both changed how integrations authenticate.
- **Licensing.** BUSL-1.1 is fine for running your own workloads but legally blocks reselling Nomad as a service. The community fork **OpenTofu** exists for Terraform; there is no equally prominent Nomad fork, so the BUSL is a real consideration for vendors.

## When to Use / When Not

**Use when:**
- You need to orchestrate a mix of containers and non-containerized apps (legacy binaries, JARs, VMs) on the same cluster.
- You want operational simplicity — a small ops team, one binary, no etcd/control-plane sprawl.
- You already run Consul and Vault, or you run on-prem / edge / bare metal where a full Kubernetes platform is overkill.
- You need multi-region federation as a built-in, not a bolt-on.

**Avoid when:**
- You want the large managed-service and operator ecosystem (Helm charts, operators, CRDs) — Kubernetes wins decisively on breadth.
- Your team's skills, tooling, and hiring pipeline are already Kubernetes-shaped.
- You need turnkey networking/ingress/RBAC without assembling companion tools.
- The BUSL license or single-vendor (IBM/HashiCorp) governance is a procurement blocker.

## Alternatives

- kubernetes/kubernetes — use instead when you need the largest ecosystem, managed offerings (EKS/GKE/AKS), and operators, and can absorb the operational complexity.
- apache/mesos — use instead for very large, heterogeneous datacenter scheduling; largely legacy now, mostly superseded.
- docker/compose — use instead for single-host, developer-scale container orchestration without a cluster.
- containerd/nerdctl — use instead when you only need a container runtime, not a scheduler.
- hashicorp/waypoint — complementary, not a substitute: an app-delivery layer that can target Nomad.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2015-09 | Initial release, announced at HashiConf[^1]. |
| 0.5 | 2016-11 | Deep Consul integration. |
| 0.8 | 2018-04 | Node drain, rolling upgrades. |
| 0.9 | 2019-04 | Pluggable task/device drivers, affinity and spread. |
| 0.10 | 2019-11 | Consul Connect service-mesh integration. |
| 0.11 | 2020-04 | CSI storage plugins (beta). |
| 1.0 | 2020-12 | GA milestone; namespaces made free[^7]. |
| 1.3 | 2022-05 | Native service discovery — Consul no longer required. |
| 1.4 | 2022-10 | Variables (built-in encrypted KV store). |
| 1.6 | 2023-07 | Node pools. |
| 1.7 | 2023-12 | Workload identity (JWT) for Consul/Vault. |
| 1.11 | 2026 | Current release line[^8]. |

## References

[^1]: HashiCorp, "Nomad Announcement" — 2015-09-28. https://www.hashicorp.com/blog/nomad
[^2]: HashiCorp, "HashiCorp adopts Business Source License" — 2023-08-10. https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license
[^3]: Nomad docs, "Architecture — Consensus (Raft)". https://developer.hashicorp.com/nomad/docs/architecture
[^4]: Nomad docs, "Scheduling — internals and optimistic concurrency". https://developer.hashicorp.com/nomad/docs/concepts/scheduling/scheduling
[^5]: Nomad docs, "Variables". https://developer.hashicorp.com/nomad/docs/concepts/variables
[^6]: HashiCorp, "Nomad scales to 2 million containers (C2M)" — 2020. https://www.hashicorp.com/blog/c2m-2-million-containers-with-hashicorp-nomad
[^7]: HashiCorp, "Announcing HashiCorp Nomad 1.0" — 2020-12. https://www.hashicorp.com/blog/announcing-hashicorp-nomad-1-0-general-availability
[^8]: Nomad releases. https://github.com/hashicorp/nomad/releases

## Tags

go, orchestration, scheduler, containers, hashicorp, devops, infrastructure, cluster-management, workload-orchestrator, raft, busl
