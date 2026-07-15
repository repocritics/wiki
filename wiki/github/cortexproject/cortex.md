# cortexproject/cortex

> Horizontally scalable, multi-tenant long-term storage for Prometheus, built as a set of stateless microservices over object storage.

[GitHub repo](https://github.com/cortexproject/cortex) ·
[Official website](https://cortexmetrics.io/) ·
[License: Apache-2.0](https://github.com/cortexproject/cortex/blob/master/LICENSE)

## Overview

Cortex is a distributed system that turns single-node Prometheus into a horizontally scalable, highly available, multi-tenant metrics store. It was started in June 2016 by Tom Wilkie and Julius Volz under the codename "Project Frankenstein," incubated at Weaveworks and later Grafana Labs, and is now a CNCF project[^1]. Applications push Prometheus (or OpenTelemetry) metrics via remote-write; Cortex replicates the samples across a cluster, flushes them to an object store (S3, GCS, Azure Blob, Swift) for effectively unlimited retention, and serves them back through a PromQL-compatible query API. A single cluster isolates many tenants by an `X-Scope-OrgID` header.

The defining tension around Cortex is organizational, not technical. Grafana Labs was for years its primary sponsor, then in 2022 forked the codebase into Grafana Mimir and relicensed that fork under AGPLv3[^2]. Most of the well-known Cortex maintainers moved with the fork and are now listed as emeritus[^3]. Cortex remains under CNCF governance and the permissive Apache-2.0 license, but you are choosing a project whose most active downstream is a competing product. It is still maintained and released, but evaluate the contributor bus factor before committing a large deployment.

The second tension is inherent to the design: Cortex trades single-binary simplicity for scale. Running it well means operating roughly a dozen distributed components, a hash ring, and an object store. For small setups this is disproportionate overhead; the payoff only appears at high cardinality and long retention.

## Getting Started

```bash
# Single-process "monolithic" mode for evaluation (filesystem backend).
docker run -p 9009:9009 \
  -v "$(pwd)/cortex-config.yaml:/etc/cortex/config.yaml" \
  quay.io/cortexproject/cortex:latest \
  -config.file=/etc/cortex/config.yaml
```

Point Prometheus at it via remote-write; the tenant is set by a header:

```yaml
# prometheus.yml
remote_write:
  - url: http://localhost:9009/api/v1/push
    headers:
      X-Scope-OrgID: tenant-a          # required: names the tenant
```

Query it as if it were Prometheus (same header selects the tenant):

```bash
curl -H 'X-Scope-OrgID: tenant-a' \
  'http://localhost:9009/prometheus/api/v1/query?query=up'
```

Production deploys use the Helm chart or jsonnet in microservices mode, not the single container above.

## Architecture / How It Works

Cortex is a single Go binary that can run every component in one process (`-target=all`, monolithic mode) or be split so each component scales independently (microservices mode). The write and read paths are separate.

Write path:

- **Distributor** — stateless front door for remote-write. Validates samples, then uses a consistent hash ring to shard each series to a set of ingesters (default replication factor 3), so a series always lands on the same ingesters.
- **Ingester** — holds recent samples in memory in a per-tenant Prometheus TSDB head, and periodically ships completed blocks to the object store. Ingesters are the only stateful hot-path component; losing a quorum of them loses recent unflushed data, which is why replication and graceful handover on rollout matter.

Read path:

- **Querier** — executes PromQL by fetching recent data from ingesters and older data from the object store via store-gateways.
- **Store-gateway** — indexes and serves the historical blocks in object storage.
- **Query-frontend / query-scheduler** — split large queries by time, cache results, and queue work fairly across tenants so one heavy tenant cannot starve others.
- **Compactor** — merges and deduplicates blocks in object storage to keep query fan-out and storage cost down.
- **Ruler** and **Alertmanager** — evaluate recording/alerting rules and handle alert routing, per tenant.

The ring is the load-bearing abstraction: membership and token ownership are coordinated through a key-value store — memberlist gossip (the recommended default), Consul, or etcd[^4]. Cortex adopted the Prometheus TSDB blocks format (sharing lineage and code with Thanos) and deprecated its original chunks storage engine, which was removed in the 1.x line[^5]. Everything durable lives in the object store; the compute tier is meant to be disposable.

## Production Notes

- **Object storage is the source of truth and the cost center.** Query latency and bill are dominated by how many blocks the store-gateways and compactor must touch. Under-provisioning the compactor lets blocks pile up and slowly degrades query performance — a common silent failure.
- **Ingesters are the fragile part.** Rollouts must hand off the in-memory head; an ungraceful restart of several ingesters at once can drop recent samples despite replication. Zone-aware replication and `PodDisruptionBudget`s are effectively mandatory on Kubernetes.
- **Cardinality is the real scaling limit.** Cost scales with the number of active series (label combinations), not raw sample volume. Per-tenant limits (max series, ingestion rate, query length) exist because one tenant's runaway cardinality can OOM ingesters cluster-wide.
- **Choose the KV store deliberately.** memberlist removes an external Consul/etcd dependency but has its own gossip convergence behavior at large cluster sizes; ring instability shows up as phantom unhealthy instances.
- **Operational surface is large.** Dozens of components, each with its own flags, resource profile, and failure mode. Expect to run the jsonnet/Helm mixin dashboards and alerts; hand-rolling observability for Cortex is a project in itself.
- **Managed escape hatch.** Amazon Managed Service for Prometheus (AMP) is built on Cortex, so its API is a way to get the model without operating the cluster[^6].

## When to Use / When Not

**Use when:**
- You have many teams/tenants and want one Prometheus-compatible backend with per-tenant isolation and quotas.
- You need long or unbounded retention on cheap object storage rather than local disk.
- Your metric volume has outgrown a single Prometheus and you already run Kubernetes and an object store.

**Avoid when:**
- You have one team and moderate volume — vanilla Prometheus (with Thanos or VictoriaMetrics if you need remote storage) is far less to operate.
- You lack the ops capacity to run a large distributed system and its object store.
- You want the most actively developed lineage — much of that energy now lives in the Grafana Mimir fork.

## Alternatives

- grafana/mimir — the AGPLv3 fork of Cortex by its former core maintainers; more features and faster development, but relicensed and single-vendor. Use when you want the current mainline of this design and AGPL is acceptable.
- thanos-io/thanos — shares the blocks storage lineage but uses a sidecar/global-query model instead of remote-write ingestion. Use when you want to keep Prometheus servers authoritative and add global query plus long-term storage.
- VictoriaMetrics/VictoriaMetrics — single-binary or clustered TSDB with much lower operational overhead and higher per-node efficiency. Use when you want scale without running a dozen microservices.
- prometheus/prometheus — the thing Cortex scales. Use when a single node's retention and throughput are enough.
- grafana/loki — sibling project reusing much of Cortex's architecture for logs. Use for logs rather than metrics.

## History

| Version | Date | Notes |
|---------|------|-------|
| Project Frankenstein | 2016-06 | Started by Tom Wilkie and Julius Volz; multi-tenant scale-out Prometheus[^1]. |
| CNCF Sandbox | 2018-09 | Accepted into the CNCF[^1]. |
| Blocks storage GA | 2020-10 | TSDB blocks engine (Thanos-lineage) becomes recommended over chunks[^5]. |
| 1.0 | 2020 | First stable release; API and storage stabilized. |
| Mimir fork | 2022-03 | Grafana Labs forks Cortex into Grafana Mimir under AGPLv3[^2]. |
| 1.x (ongoing) | 2026 | Active under CNCF; last push 2026-07 per repository metadata. |

## References

[^1]: Cortex README, "History of Cortex" and CNCF project page. https://github.com/cortexproject/cortex and https://www.cncf.io/projects/cortex/
[^2]: Grafana Labs, "Announcing Grafana Mimir" — 2022-03-30 (fork of Cortex, AGPLv3). https://grafana.com/blog/2022/03/30/announcing-grafana-mimir/
[^3]: Cortex README, "Emeritus Maintainers." https://github.com/cortexproject/cortex#emeritus-maintainers
[^4]: Cortex architecture documentation (ring, key-value store). https://cortexmetrics.io/docs/architecture/
[^5]: Grafana Labs, "Now GA: Cortex blocks storage" — 2020-10-06. https://grafana.com/blog/2020/10/06/now-ga-cortex-blocks-storage-for-running-prometheus-at-scale-with-reduced-operational-complexity/
[^6]: Amazon Managed Service for Prometheus (built on Cortex). https://aws.amazon.com/prometheus/

## Tags

go, prometheus, observability, metrics, time-series, monitoring, cncf, multi-tenant, distributed-systems, kubernetes, long-term-storage, object-storage
