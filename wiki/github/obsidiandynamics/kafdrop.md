# obsidiandynamics/kafdrop

> A read-mostly web UI for inspecting Kafka clusters — brokers, topics, partitions, consumer groups, and messages — packaged as a single Spring Boot app.

[GitHub repo](https://github.com/obsidiandynamics/kafdrop) ·
No official website ·
[License: Apache-2.0](https://github.com/obsidiandynamics/kafdrop/blob/master/LICENSE)

## Overview

Kafdrop is a browser-based viewer for Apache Kafka. It renders cluster metadata (brokers, controller status, topic/partition assignments, replication state) and lets you browse messages and consumer-group offsets/lag without a CLI. It is the most widely deployed open-source Kafka UI that ships as a plain JAR or a small Docker image, which is the main reason it persists in a crowded field: it does one job, has almost no operational surface, and requires no backing database of its own.

The project is a rewrite. The original Kafdrop was a HomeAdvisor tool built on JSP and dependent on ZooKeeper. Emil Koutanov (obsidiandynamics) reimplemented it as Kafdrop 3 on Spring Boot, and that fork is now the canonical, actively maintained line[^1]. The rewrite is the reason nearly all current documentation, config, and Docker images live under `obsidiandynamics/kafdrop` rather than the older HomeAdvisor namespace.

Its defining tension is scope. Kafdrop is deliberately a *viewer*, not a control plane: it is stateless, holds no auth, and its write features (create/delete topic, produce message) are opt-in and mostly discouraged in production. Teams that want RBAC, audit trails, or multi-cluster management outgrow it and move to Kafka UI (provectus), Redpanda Console, or a commercial control center. Teams that want "just show me the topics" keep it for years.

## Getting Started

```sh
docker run -d --rm -p 9000:9000 \
    -e KAFKA_BROKERCONNECT=broker1:9092,broker2:9092 \
    obsidiandynamics/kafdrop
```

Then open `http://localhost:9000`. Running the JAR directly is equivalent (Java 17+ required):

```sh
java --add-opens=java.base/sun.nio.ch=ALL-UNNAMED \
    -jar target/kafdrop-<version>.jar \
    --kafka.brokerConnect=localhost:9092
```

`kafka.brokerConnect` defaults to `localhost:9092`. As of Kafdrop 3.10.0 no ZooKeeper connection is needed — all cluster info comes from the Kafka admin API[^2]. Message deserialization can be set globally or per-topic with `--message.format=AVRO|PROTOBUF|DEFAULT`, optionally wired to a Schema Registry via `--schemaregistry.connect`.

## Architecture / How It Works

Kafdrop is a single Spring Boot (embedded Tomcat) web application. It is stateless: it maintains no database and persists nothing about your cluster. Every page is rendered from live calls made through the standard Kafka `AdminClient` and consumer APIs, so what you see is always the current cluster state, and Kafdrop itself has nothing to back up or migrate.

Key internals worth knowing:

- **HTML and JSON share the same routes.** Any HTML view can be returned as JSON by sending `Accept: application/json`; an OpenAPI spec is served at `/v3/api-docs` and a Swagger UI at `/swagger-ui.html`. This makes Kafdrop usable as an ad-hoc read API, not just a UI.
- **Message browsing is a bounded consumer read.** Viewing messages spins up a consumer against a partition and reads a window of records; it is not a search index. Deserializers (String, JSON, Avro, Protobuf) are selected per-view, with Avro/Protobuf resolved via Schema Registry or a mounted `.desc` descriptor directory.
- **Broker connection is delegated to the Kafka client.** Security is configured by handing Kafdrop a `kafka.properties` file plus optional JKS trust/keystores; SASL (SCRAM, PLAIN, and via mounted extra JARs, AWS MSK IAM) and TLS are supported because Kafdrop passes them straight through to the underlying client. Azure Event Hubs works through the same Kafka-protocol path.
- **Actuator surface.** Spring Actuator health/info endpoints are exposed under `/actuator`, which is what you point Kubernetes liveness/readiness probes at.

Because the app is thin over the Kafka client, its capabilities track what the admin/consumer APIs expose — it can show ACLs and create topics, but it is not a substitute for `kafka-acls` or a schema governance tool.

## Production Notes

- **No authentication, by design.** Kafdrop has no built-in login. The maintainers' documented approach is to put a reverse proxy (the README shows an NGINX Basic Auth config) or an SSO gateway in front[^3]. Exposing Kafdrop directly on a routable network hands anyone read access to every message in the cluster — and, if write flags are on, the ability to delete topics.
- **Write features are footguns.** `topic.deleteEnabled` and `topic.createEnabled` default to on; `message.sendEnabled` defaults to off. In shared or production clusters, explicitly set `--topic.deleteEnabled=false` (and consider `--topic.createEnabled=false`) so a curious click can't drop a topic.
- **CORS is permissive out of the box.** Since the 2.x API line, CORS headers are sent for all endpoints with `cors.allowOrigins=*` and `cors.allowCredentials=true` by default. Lock these down or set `cors.enabled=false` if the JSON API is reachable from browsers.
- **It reads real data from real brokers.** Browsing a high-throughput partition, or a topic with very large messages, makes Kafdrop consume those records live; keep the JVM heap sane (`JVM_OPTS="-Xms32M -Xmx64M"` is a common floor) and expect memory/latency to scale with message size, not just count.
- **One UI, one cluster.** A Kafdrop instance connects to a single cluster. Multi-cluster shops run one instance per cluster (often one Deployment each in Kubernetes) rather than switching within the UI.
- **Upgrade friction is mostly the JVM.** Kafdrop 4.x moved to Spring Boot 3 and requires Java 17+; older deployments pinned to Java 8/11 images need a base-image bump. The `--add-opens=java.base/sun.nio.ch=ALL-UNNAMED` flag is needed on modern JDKs for the JAR path.

## When to Use / When Not

**Use when:**
- You want a fast, low-footprint way to inspect topics, offsets, and lag during development or incident triage.
- You need a drop-in Docker/Helm UI with no database and near-zero config.
- You want a JSON/OpenAPI read view of cluster state for scripts or dashboards.

**Avoid when:**
- You need authentication, RBAC, or per-user audit logging natively (Kafdrop has none).
- You want a management/control plane: schema governance, connector management, reassignment, quotas.
- You need message search, historical retention of views, or multi-cluster switching in one pane.

## Alternatives

- provectus/kafka-ui — richer management UI with auth/RBAC and multi-cluster; use when you need governance, not just viewing.
- redpanda-data/console — polished console (works with Kafka and Redpanda); use when you want a more feature-complete, actively funded UI.
- yahoo/CMAK — ZooKeeper-era cluster manager; use only for legacy ZooKeeper-based operational tasks.
- confluentinc/cp-kafka (Control Center) — commercial, enterprise governance; use when you're already on Confluent Platform.
- linkedin/kafka-tools / CLI — use when you want scriptable, no-UI operations over a web app.

## History

| Version | Date | Notes |
|---------|------|-------|
| kafdrop-2.0.0 | (HomeAdvisor era) | Original JSP + ZooKeeper tool, later forked[^1]. |
| 3.10.0 | 2019-09-27 | ZooKeeper dependency dropped; cluster info via admin API[^2]. |
| 3.30.0 | 2022-04-09 | Late 3.x line (Avro/Protobuf, Schema Registry, JSON API mature). |
| 3.31.0 | 2023-03-20 | Final 3.x release. |
| 4.0.0 | 2023-10-09 | Spring Boot 3 / Java 17 baseline[^4]. |
| 4.1.0 | 2024-12-10 | 4.x maintenance. |
| 4.2.0 | 2025-07-31 | Latest release at time of writing[^4]. |

## References

[^1]: obsidiandynamics/kafdrop — README, project history and lineage from the HomeAdvisor original. https://github.com/obsidiandynamics/kafdrop
[^2]: Kafdrop README, "As of Kafdrop 3.10.0, a ZooKeeper connection is no longer required." Release 3.10.0 published 2019-09-27. https://github.com/obsidiandynamics/kafdrop/releases/tag/3.10.0
[^3]: Kafdrop README, "Securing the Kafdrop UI" — NGINX Basic Auth workaround; no native auth. https://github.com/obsidiandynamics/kafdrop#securing-the-kafdrop-ui
[^4]: Kafdrop GitHub releases — 4.0.0 (2023-10-09) through 4.2.0 (2025-07-31). https://github.com/obsidiandynamics/kafdrop/releases

## Tags

java, kafka, kafka-ui, web-ui, spring-boot, observability, event-streaming, docker, kubernetes, devops-tools
