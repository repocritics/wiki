# provectus/kafka-ui

> A web UI for inspecting and managing Apache Kafka clusters — now frozen at Provectus and continued by a community fork.

[GitHub repo](https://github.com/provectus/kafka-ui) ·
[Docs](https://docs.kafka-ui.provectus.io/) ·
[License: Apache-2.0](https://github.com/provectus/kafka-ui/blob/master/LICENSE)

## Overview

kafka-ui (marketed as "UI for Apache Kafka") is a self-hosted web console for Kafka. It lets you browse brokers, topics, partitions, consumer groups and their lag, produce and read messages (JSON, plain text, Avro, Protobuf), manage Schema Registry entries, inspect Kafka Connect connectors, and run ksqlDB queries — all from a browser instead of `kafka-*.sh` CLI scripts[^1]. It targets developers and operators who want a read-mostly observability pane over one or several clusters without paying for a commercial control plane.

The important thing to know before adopting it: **the Provectus repository is effectively dormant.** The last commit landed in July 2024, releases stopped at the 0.7.x line, and open development moved to a community fork, **kafbat/kafka-ui**, formed by former maintainers[^2]. The Provectus repo still carries ~12k stars and remains the name most people search for, but new fixes, Kafka-version compatibility, and CVE patches ship under kafbat, not here. Treat this page's subject as a stable-but-archived snapshot and the fork as the living project.

The defining tradeoff is scope-by-design: kafka-ui deliberately stays a thin, stateless viewer. It stores nothing itself (no database), which keeps deployment trivial, but also means it has no historical metrics, no alerting, and no audit trail beyond what it reads live from the cluster.

## Getting Started

The fastest path is the prebuilt Docker image with the config wizard enabled:

```bash
docker run -it -p 8080:8080 \
  -e DYNAMIC_CONFIG_ENABLED=true \
  provectuslabs/kafka-ui
# open http://localhost:8080 and add a cluster via the UI wizard
```

For a persistent, declaratively-configured install, mount a YAML file or use environment variables:

```yaml
# docker-compose.yml
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - 8080:8080
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
```

Cluster settings follow the `KAFKA_CLUSTERS_<n>_*` pattern; the same keys exist in the YAML config file under `kafka.clusters[]`.

## Architecture / How It Works

kafka-ui is a single deployable unit: a Spring Boot backend written in Java, built on Spring WebFlux (the reactive, non-blocking stack), that also serves a compiled React + TypeScript single-page app from the same JAR / container[^1]. There is no separate frontend service and no datastore.

The backend talks to Kafka through the standard Java client libraries — `AdminClient` for cluster/topic/config metadata, consumer and producer APIs for message browse and produce. Auxiliary integrations (Schema Registry, Kafka Connect, ksqlDB, Prometheus-style JMX metrics) are separate HTTP clients wired per-cluster from config. Each configured cluster is polled independently; the UI is a projection of whatever the backend can currently reach.

Configuration is fully external and static-by-default: YAML file plus `KAFKA_CLUSTERS_*` environment overrides, resolved at boot. Setting `DYNAMIC_CONFIG_ENABLED=true` opts into a mutable mode where clusters added through the in-browser wizard are written back to a config file on disk — convenient for demos, but it means UI actions now mutate deployment state, which some operators disable in production.

Extensibility is concentrated in the **serde** layer: custom serialization/deserialization plugins let the UI decode proprietary message formats (AWS Glue, Smile, or hand-written serdes) beyond the built-in Avro/Protobuf/JSON handling. Access control is bolted on above the read layer — optional OAuth 2.0 (GitHub/GitLab/Google) for authentication, plus a role-based access control (RBAC) system and a data-masking feature to obfuscate sensitive fields in message payloads.

## Production Notes

- **Maintenance status is the first-order concern.** With the Provectus repo frozen since mid-2024, pinning `provectuslabs/kafka-ui:latest` gives you a build that will not receive Kafka-compatibility or security updates. Teams running this in production should either migrate to kafbat/kafka-ui or accept the freeze consciously and pin to a specific digest.
- **It is a viewer, not a monitoring system.** No metrics are persisted, so there is no lag history, no trend graphs over time, and no alerting. Pair it with Prometheus/Grafana or a commercial tool if you need those. The "metrics dashboard" reflects live JMX reads only.
- **Message browsing on large topics is expensive.** Reading messages spins up real consumers; scanning or filtering across high-volume topics/partitions consumes broker and network resources and can be slow. It is an inspection tool, not a search engine — expect to seek by offset/timestamp rather than full-scan.
- **Auth is opt-in and easy to forget.** A default `docker run` exposes an unauthenticated UI with full produce/consume and topic-admin power to anyone who can reach port 8080. Never expose it to a network without enabling OAuth + RBAC first.
- **Memory footprint scales with brokers/partitions.** The backend caches cluster metadata; very large clusters (thousands of topics/partitions) push JVM heap usage up, and the default container limits may need raising.
- **`DYNAMIC_CONFIG_ENABLED` blurs config boundaries.** Enabling it lets the UI rewrite its own config file; in immutable-infrastructure setups this is usually left off so the deployment stays the single source of truth.

## When to Use / When Not

**Use when:**
- You want a free, self-hosted, single-container UI to inspect topics, messages, consumer lag, and schemas.
- Your team is CLI-averse and needs an approachable pane for day-to-day Kafka debugging.
- You need custom serde decoding or per-message data masking without a commercial license.

**Avoid when:**
- You need an actively maintained project with a security-patch cadence — use kafbat/kafka-ui instead, which is the same codebase under continued development.
- You need historical metrics, alerting, or an audit log — this stores no state.
- You need a governed control plane (approvals, lineage, multi-tenant quotas) — reach for a managed/commercial offering.

## Alternatives

- kafbat/kafka-ui — the direct community continuation of this exact codebase; use it instead when you want ongoing maintenance and Kafka-version updates.
- obsidiandynamics/kafdrop — lighter, older read-only-leaning topic/message viewer; use when you want the smallest possible footprint and don't need write/admin features.
- redpanda-data/console — polished UI (works against Kafka too, not just Redpanda); use when you want a more actively developed open-core console.
- Conduktor — commercial desktop/web Kafka platform; use when you need governance, testing, and enterprise support rather than a bare viewer.
- provectus itself also ships schema/connect features that AKHQ (tchiotludo/akhq) overlaps with; use AKHQ when you prefer a config-file-first, LDAP-friendly alternative.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-11-26 | Repository created at Provectus[^3]. |
| 0.x | 2020–2022 | Iterative growth: message browse, Schema Registry, Connect, ksqlDB integrations. |
| 0.7.x | 2023 | Last Provectus release line; RBAC, data masking, serde plugins mature[^1]. |
| — | 2024 (early) | Maintainers spin up kafbat/kafka-ui as the community continuation[^2]. |
| — | 2024-07-26 | Final commit to the Provectus repo; development effectively frozen[^3]. |

## References

[^1]: Project README and features list, provectus/kafka-ui. https://github.com/provectus/kafka-ui
[^2]: kafbat/kafka-ui — community-maintained continuation of UI for Apache Kafka. https://github.com/kafbat/kafka-ui
[^3]: GitHub repository metadata (created 2019-11-26, last push 2024-07-26), fetched via GitHub API. https://github.com/provectus/kafka-ui

## Tags

java, spring-boot, apache-kafka, kafka, web-ui, event-streaming, cluster-management, schema-registry, kafka-connect, self-hosted, observability, dormant
