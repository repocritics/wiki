# emqx/emqx

> A distributed MQTT broker written in Erlang/OTP, built for very large device fleets — now under a Business Source License with a cluster gate.

[GitHub repo](https://github.com/emqx/emqx) ·
[Official website](https://www.emqx.com) ·
[License: BSL 1.1](https://github.com/emqx/emqx/blob/master/LICENSE) (GitHub reports NOASSERTION)

## Overview

EMQX is an MQTT message broker: clients connect over MQTT 5.0 / 3.1.1 / 3.1 (and MQTT-SN, CoAP, LwM2M, MQTT over QUIC via gateways), publish to topics, and subscribe to topic filters, with the broker handling routing, retained messages, sessions, QoS, and access control. It targets the high end of the scale range — device fleets in the hundreds of thousands to millions per cluster — and positions itself against lighter single-node brokers like Mosquitto. The project began as `emqttd` and was renamed EMQX; it has been developed by EMQ (Hangzhou) since 2012[^1].

The defining fact about EMQX in 2026 is its license. Through the 4.x and early 5.x lines the open-source broker was Apache 2.0. Starting with **v5.9.0**, EMQX merged its former Open Source and Enterprise editions into one codebase under the **Business Source License (BSL) 1.1**, and — critically — running a cluster of more than one node now requires a license file[^2]. A single node runs freely; the moment you add a second node for high availability, you are in license-required territory. BSL source is available and converts to Apache 2.0 after a change date, but "source-available" is not OSI-open-source, and this shift is the single most important thing to understand before adopting current EMQX.

The tension, then: EMQX is the most feature-complete self-hostable MQTT broker (rule engine, 50+ data bridges, durable sessions, clustering that scales past Mnesia's limits), but the versions that carry those features are no longer freely clusterable. Teams that want Apache-2.0 clustering are pinned to pre-5.9 releases or looking at alternatives.

## Getting Started

```bash
# Single node via Docker (image is now the unified "enterprise" image)
docker run -d --name emqx \
  -p 1883:1883 -p 8083:8083 -p 8084:8084 \
  -p 8883:8883 -p 18083:18083 \
  emqx/emqx-enterprise:latest
# Dashboard: http://localhost:18083  (default admin / public)
```

```bash
# Publish/subscribe with the mosquitto CLI against the broker
mosquitto_sub -h localhost -p 1883 -t 'sensors/#' -q 1 &
mosquitto_pub -h localhost -p 1883 -t 'sensors/room1/temp' -m '21.4' -q 1
```

The port map matters: `1883` MQTT, `8883` MQTT/TLS, `8083` MQTT-over-WebSocket, `8084` WSS, `18083` the dashboard and HTTP API.

```bash
# Everything the dashboard does is also available over the HTTP API
curl -u admin:public http://localhost:18083/api/v5/clients
curl -u admin:public http://localhost:18083/api/v5/stats
```

A rule that reforwards hot readings and pushes them to a webhook is a handful of lines of SQL bound to an action, defined either in the dashboard or in `emqx.conf`:

```
SELECT payload.temp AS t, topic
FROM  "sensors/#"
WHERE t > 30
```

## Architecture / How It Works

EMQX runs on the Erlang BEAM VM, which is the reason it exists in the form it does: each MQTT connection is a lightweight Erlang process, and the scheduler multiplexes millions of them across OS threads. Backpressure, per-connection isolation, and hot code paths are all inherited from OTP rather than hand-built.

- **Routing and clustering.** The broker maintains a distributed routing table mapping topic filters to subscriber nodes. Classic clustering used Mnesia (Erlang's built-in distributed DB), whose full-mesh replication practically caps at a handful of nodes. EMQX 5.0 introduced **Mria**, an extension that adds a **core / replicant** topology: core nodes hold the authoritative state, replicant nodes stream changes and carry connections, letting a cluster scale horizontally past the old Mnesia ceiling[^3].
- **Rule Engine.** An SQL-like language selects and transforms in-flight messages (`SELECT payload.temp AS t FROM "sensors/#" WHERE t > 30`) and routes results to actions. Actions feed **data bridges / integrations** — Kafka, PostgreSQL, MySQL, MongoDB, Redis, InfluxDB, MQTT-to-MQTT, HTTP webhooks, and cloud sinks — so EMQX often functions as the ingest-and-fan-out tier of an IoT pipeline, not just a broker.
- **Sessions.** Persistent MQTT sessions traditionally lived in memory. Newer 5.x lines add **durable sessions**, persisting session and message state (RocksDB-backed) so a subscriber can disconnect and later receive QoS 1/2 messages that arrived while it was offline, surviving node restarts.
- **Auth / ACL.** Authentication (password, JWT, PSK, X.509, LDAP, external SQL/Redis lookups) and authorization (topic ACLs) are pluggable chains evaluated per connect/publish/subscribe.
- **Extensibility.** A plugin system and lifecycle **hooks** let you intercept the connection/message lifecycle; ExHook exposes hooks over gRPC for non-Erlang code.

The whole system is configured through `emqx.conf` (HOCON format) plus runtime overrides, and managed through the dashboard, a rich HTTP API, or the `emqx ctl` CLI.

## Production Notes

- **The cluster license gate (v5.9.0+).** Any deployment past a single node needs a license file loaded, or the cluster will not form. Plan license acquisition into your HA rollout; do not discover this in an incident. Pre-5.9 Apache-2.0 releases have no such gate but miss later features and fixes.
- **You are running Erlang in production.** Deep debugging means `emqx remote_console`, `observer`, and OTP knowledge. Split-brain, Mnesia/Mria inconsistency, and cluster partition healing are Erlang-distribution problems; teams without any BEAM familiarity hit a real operational learning curve.
- **Tuning for scale is not free.** Millions of connections require OS-level work: file-descriptor limits, TCP buffer and backlog tuning, ephemeral port ranges, and Erlang VM flags (scheduler and port counts). The advertised "100M connections" is a multi-node cluster figure, not a single box.
- **Cluster discovery.** Node discovery via static list, DNS, etcd, or the Kubernetes operator each has failure modes; the `emqx-operator` is the supported path on K8s and is where most Helm-era footguns have been smoothed.
- **Upgrades are matrix-constrained.** Rolling-upgrade support between minor versions is explicitly tabulated in the README, with real caveats: pre-5.4 routing tables are dropped (upgrade to 5.9 first, then a full-cluster — not rolling — restart before 5.10), old limiter config must be removed before some steps, and **durable session state is lost on the v5→v6 upgrade** (clients reconnect into clean sessions). Read the upgrade matrix before every jump.
- **Dashboard defaults.** The default `admin/public` credential is a standing footgun; change it before exposing `18083`.

## When to Use / When Not

**Use when:**
- You need MQTT at fleet scale (hundreds of thousands+ concurrent clients) with clustering, durable sessions, and built-in data integrations.
- You want an SQL rule engine and 50+ sink connectors in the broker instead of bolting on a separate stream processor.
- MQTT 5.0 features (shared subscriptions, topic aliases, message expiry) and protocol breadth (QUIC, CoAP, LwM2M) are requirements.

**Avoid when:**
- You need OSI-approved open-source clustering: current EMQX gates multi-node behind a BSL license — look at VerneMQ or pinned pre-5.9 EMQX.
- You want a tiny, single-binary edge broker with minimal ops: Mosquitto or NanoMQ are far lighter.
- Your team has zero appetite for Erlang/OTP operational concepts.

## Alternatives

- eclipse/mosquitto — small C broker, EPL/EDL licensed. Use instead when you want a single lightweight node at the edge and don't need clustering.
- vernemq/vernemq — Erlang-based distributed MQTT broker, Apache 2.0. Use instead when you want clustered MQTT without a BSL license gate.
- emqx/nanomq — EMQ's own ultralight MQTT broker for edge/gateway devices. Use instead when footprint matters more than the rule engine and integrations.
- hivemq/hivemq-community-edition — Java MQTT broker. Use instead in JVM shops or when HiveMQ's enterprise support model fits.
- nats-io/nats-server — not MQTT-native (has an MQTT gateway), Apache 2.0. Use instead when your workload is general pub/sub and messaging rather than device-protocol MQTT.

## History

| Version | Date | Notes |
|---------|------|-------|
| emqttd 1.0 | 2016 | Early Erlang MQTT broker, later renamed EMQX[^1]. |
| 4.0 | 2020 | 4.x line; Apache 2.0 open-source broker, rule engine matured. |
| 5.0 | 2022 | Major rewrite: Mria core/replicant clustering, MQTT over QUIC, new config (HOCON)[^3]. |
| 5.9.0 | 2025 | Editions unified under BSL 1.1; multi-node clustering now requires a license[^2]. |
| 6.x | 2025–2026 | Current major line; v5→v6 upgrade drops durable session state. |

## References

[^1]: EMQX / EMQ company and project background. https://www.emqx.com/en/about
[^2]: "EMQX Adopts Business Source License" and License FAQ — BSL 1.1 from v5.9.0, clustering requires a license file. https://www.emqx.com/en/news/emqx-adopts-business-source-license
[^3]: Mria core/replicant clustering (EMQX 5.0). https://github.com/emqx/mria

## Tags

erlang, mqtt, mqtt-broker, iot, message-broker, pubsub, messaging, distributed-systems, bsl-license, edge-computing, iiot
