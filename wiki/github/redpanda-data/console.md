# redpanda-data/console

> A web UI for inspecting, debugging, and managing Kafka and Redpanda clusters — the tool formerly known as Kowl.

[GitHub repo](https://github.com/redpanda-data/console) ·
[Official website](https://redpanda.com) ·
License: BSL 1.1 (Business Source License, converts to Apache-2.0 at Change Date)[^2]

## Overview

Redpanda Console is a browser-based control panel for Apache Kafka and Redpanda clusters: it lists topics and brokers, browses and filters messages, manages consumer groups and their offsets, inspects a Schema Registry, drives Kafka Connect clusters, and manages ACLs and SASL/SCRAM users[^1]. It started life as **Kowl**, an independent project by CloudHut; Redpanda acquired it and rebranded it to Redpanda Console in 2022. The README still carries the "previously known as Kowl" line[^1].

The backend is Go and the frontend is React/TypeScript; a release build embeds the compiled frontend into a single static binary (also shipped as a Docker image and a Helm chart), so a deployment is one process pointed at a cluster[^1]. Console is read-heavy by design — it is an observability and light-administration surface, not a data pipeline component. It sits beside the cluster and speaks the Kafka protocol plus the various HTTP admin APIs (Schema Registry, Connect REST, Redpanda Admin API).

The defining tension is licensing. Despite the "developer-friendly UI" framing and a public repo, Console is **not** OSI open source: the core is under the Business Source License, and a set of enterprise features (notably SSO/OIDC login and role-based access control) is gated behind the separate Redpanda Community License and an enterprise license key[^2][^3]. Anyone deploying Console past a single trusted operator needs to know which capabilities require a paid license before standardizing on it.

## Getting Started

Docker against a local broker (Console runs in its own network namespace, so `host.docker.internal` is used as the bootstrap address on macOS/Windows)[^1]:

```shell
docker run -p 8080:8080 \
  -e KAFKA_BROKERS=host.docker.internal:9092 \
  docker.redpanda.com/redpandadata/console:latest
```

Against a SASL_SSL cluster (e.g. Confluent Cloud), everything can be supplied by env var:

```shell
docker run -p 8080:8080 \
  -e KAFKA_BROKERS=pkc-xxxxx.gcp.confluent.cloud:9092 \
  -e KAFKA_TLS_ENABLED=true \
  -e KAFKA_SASL_ENABLED=true \
  -e KAFKA_SASL_USERNAME=xxx \
  -e KAFKA_SASL_PASSWORD=xxx \
  docker.redpanda.com/redpandadata/console:latest
```

For anything non-trivial, a YAML config file is the maintainable path; env vars map onto the same config tree and are best reserved for secrets and per-environment overrides. The docs ship a `docs/local` docker-compose that boots Redpanda + Console together for evaluation[^1][^4].

## Architecture / How It Works

Console is a thin, stateless proxy over cluster APIs. It holds no database of its own — every view is assembled live from the cluster on request, which is why it starts instantly and scales horizontally, but also why very large clusters can make individual pages expensive.

- **Kafka protocol client.** The Go backend talks to brokers over the native Kafka protocol via franz-go (Redpanda's own Go client library)[^5], which is what lets it support Redpanda-specific and newer Kafka features quickly.
- **Message viewer / filters.** The headline feature is browsing topic messages with ad-hoc filters written in JavaScript, evaluated server-side in an embedded JS interpreter. Deserialization covers JSON, Avro, Protobuf, CBOR, XML, MessagePack, text, and binary; encoding is auto-detected except for Protobuf and CBOR, which need explicit configuration[^1]. Avro/Protobuf decoding is resolved through the configured Schema Registry.
- **Admin surfaces.** Consumer groups, ACLs, and SASL/SCRAM users are managed over the Kafka admin protocol; Schema Registry, Kafka Connect (multiple clusters), and Redpanda-specific features (data transforms) are each reached through their respective HTTP APIs and surfaced as UI panels.
- **Frontend.** React/TypeScript, built with bun; the repo runs react-doctor in CI to catch React performance anti-patterns[^1]. In a release the built assets are embedded into the Go binary, so there is no separate static-file host to operate.

Because Console reflects cluster state rather than storing it, its correctness and performance are bounded by the cluster's own metadata and message-fetch cost, not by anything Console persists.

## Production Notes

- **Licensing is the first design decision, not an afterthought.** Login (SSO/OIDC) and RBAC are enterprise features requiring a Redpanda license[^2][^3]. Without one, Console has no built-in authentication — an unauthenticated deployment exposes topic contents, consumer-group editing, ACL management, and SCRAM user administration to anyone who can reach the port. Put it behind a reverse proxy / network policy, or budget for the enterprise license.
- **It is genuinely powerful mutation surface, not just a viewer.** Editing consumer-group offsets, deleting groups, and editing ACLs are real, destructive cluster operations exposed in a few clicks. Treat access as production-cluster access.
- **Message browsing cost scales with the cluster.** JavaScript filters run server-side and, depending on the query, can require scanning large partition ranges; broad filters on high-volume topics are slow and put fetch load on brokers. Prefer bounded time/offset ranges.
- **Protobuf/CBOR need configuration.** Auto-detection does not cover them; you must wire up proto descriptors or a schema source, or messages render as binary.
- **BSL usage restriction.** The BSL permits most use but forbids offering Console (or Redpanda) as a competing hosted service until the Change Date, when the grant converts to Apache-2.0[^2]. Internal and customer-facing operational use is fine; reselling it as a service is the excluded case.
- **Kowl heritage.** Old `cloudhut/kowl` references, images, and blog posts are stale; configuration and image coordinates changed with the rebrand. Use current Redpanda docs, not Kowl-era material.

## When to Use / When Not

**Use when:**
- You run Redpanda, or Kafka, and want a maintained UI for topic inspection, message debugging, and consumer-group/ACL management.
- You want a single-binary or single-container deployment with no backing datastore to operate.
- You need live message decoding across Avro/Protobuf/JSON via Schema Registry with expressive filtering.

**Avoid when:**
- You require OSI-approved open source or must avoid BSL/source-available terms — the license will disqualify it in some organizations.
- You need built-in auth/RBAC but cannot pay for the enterprise tier — an OSS alternative gives you that for free.
- Your cluster is not Kafka-protocol-compatible, or you need automated rebalancing/ops rather than a human-facing UI.

## Alternatives

- kafbat/kafka-ui — the actively maintained fork of provectus/kafka-ui; use when you need a fully Apache-2.0 UI with built-in auth and RBAC at no cost.
- provectus/kafka-ui — the original (now largely superseded by the kafbat fork); use only if you are already pinned to it.
- obsidiandynamics/kafdrop — lightweight, mostly read-only cluster/topic viewer; use when you want the smallest possible footprint.
- tchiotludo/akhq — feature-rich Apache-2.0 Kafka GUI with LDAP/OIDC auth; use when license and auth both matter and you want breadth.
- Confluent Control Center (proprietary) — use when you are already standardized on Confluent Platform and want vendor-integrated monitoring.

## History

| Version | Date | Notes |
|---------|------|-------|
| Kowl | 2019-09-29 | Repo created as CloudHut's Kowl[^1]. |
| Redpanda Console (rename) | 2022 | Kowl acquired by Redpanda and rebranded; BSL/RCL licensing applied[^2]. |
| v2.x | 2022–2024 | Redpanda-branded line; enterprise login/RBAC gating. |
| v3.8.0 | 2026-06-29 | Latest release at time of writing[^6]. |

## References

[^1]: Redpanda Console README. https://github.com/redpanda-data/console
[^2]: Redpanda, "Why we chose the Business Source License." https://redpanda.com/blog/open-source
[^3]: Redpanda Console licenses README (BSL core + RCL enterprise features). https://github.com/redpanda-data/console/blob/master/licenses/README.md
[^4]: Redpanda installation / quick-start docs. https://docs.redpanda.com/current/get-started/quick-start/
[^5]: franz-go — Go Kafka client maintained by Redpanda. https://github.com/twmb/franz-go
[^6]: Redpanda Console releases. https://github.com/redpanda-data/console/releases

## Tags

kafka, redpanda, kafka-ui, streaming, observability, go, typescript, react, schema-registry, kafka-connect, dataops, source-available
