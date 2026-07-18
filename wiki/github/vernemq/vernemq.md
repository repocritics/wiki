# vernemq/vernemq

> A distributed MQTT broker written in Erlang/OTP — clustering as a first-class feature, with an "open source code, paid binaries" distribution model you must understand before deploying.

[GitHub repo](https://github.com/vernemq/vernemq) ·
[Official website](https://vernemq.com) ·
[License: Apache-2.0](https://github.com/vernemq/vernemq/blob/main/LICENSE)

## Overview

VerneMQ is an MQTT broker (protocol versions 3.1, 3.1.1, and 5.0) built on Erlang/OTP, started in 2015 by Erlio GmbH[^1]. Its differentiator among open MQTT brokers is native clustering: nodes form a masterless cluster that replicates subscriber and retained-message state, so IoT fleets can spread very large connection counts across commodity machines rather than sharding at the application layer. The Erlang runtime is a natural fit — each MQTT session maps to lightweight processes, and the VM's preemptive scheduling keeps one slow client from stalling others.

The project's defining tension is distribution economics rather than technology. The source is Apache-2.0 and always has been, but since 2019 the pre-built packages and official Docker images ship under a EULA: free for evaluation and non-commercial use, paid subscription for commercial production use of the binaries[^2]. Building from source keeps you fully on Apache-2.0. The model funds maintenance but pushed part of the community toward EMQX and Mosquitto — check it first in any adoption decision.

At 3.6k stars and 427 forks it is mid-sized — far smaller than EMQX's ecosystem — but genuinely maintained: 2.1.x releases shipped through 2025, a 2.1.3 RC landed in April 2026, and commits continue on `main`[^3]. The cadence (roughly one minor release per year, small maintainer team) cuts both ways: a stable protocol surface, but slower feature and issue turnaround than commercially-staffed competitors (181 open issues at time of writing).

## Getting Started

The fastest path is Docker (note the mandatory EULA flag):

```bash
docker run -p 1883:1883 \
  -e "DOCKER_VERNEMQ_ACCEPT_EULA=yes" \
  -e "DOCKER_VERNEMQ_ALLOW_ANONYMOUS=on" \
  --name vernemq vernemq/vernemq
```

Test with any MQTT client:

```bash
mosquitto_sub -t 'sensors/#' &
mosquitto_pub -t 'sensors/room1' -m '21.5'
```

Building from source (the fully-Apache-2.0 route) requires Erlang/OTP 25–27, a C compiler, and libsnappy for the LevelDB storage backend[^1]:

```bash
git clone https://github.com/vernemq/vernemq && cd vernemq
make rel
_build/default/rel/vernemq/bin/vernemq start
vmq-admin cluster show
```

Configuration lives in `vernemq.conf`; runtime administration (cluster membership, session listing, live tracing) goes through the `vmq-admin` CLI.

## Architecture / How It Works

VerneMQ is a classic Erlang/OTP system. Each connected client is served by a pair of processes: a protocol FSM handling the MQTT wire protocol and a `vmq_queue` process owning that session's message queue. Queues survive disconnects for persistent sessions; offline messages spill to disk via LevelDB (eleveldb), which is why the build needs snappy.

Routing is local-first. Every node holds a full copy of the subscription trie and retained-message store; a publish does a local topic-trie lookup, delivers to local subscribers directly, and forwards to remote nodes with matching subscribers over inter-node cluster links. This means routing state must be replicated everywhere — done eventually-consistently through a gossip-based metadata layer. Historically this was Plumtree (epidemic broadcast trees, from the Riak lineage); a newer backend based on Server Wide Clocks (`vmq_swc`) was introduced to improve sync performance on larger clusters, and the metadata plugin is switchable — but switching requires a full cluster restart[^4].

Extensibility is hook-based. Plugins register for hooks such as `auth_on_register`, `auth_on_publish`, and `on_deliver`, and can be written as Erlang/Elixir OTP applications, as Lua scripts via the bundled `vmq_diversity` plugin (which also provides the PostgreSQL/MySQL/MongoDB/Redis/Memcached auth integrations), or as HTTP webhooks via `vmq_webhooks`[^1]. Auth is entirely plugin-land: the broker ships with file-based auth and denies anonymous access unless explicitly enabled.

## Production Notes

**Licensing is the first footgun.** The official Docker image refuses to start without `DOCKER_VERNEMQ_ACCEPT_EULA=yes`, and that EULA restricts commercial production use of the pre-built binaries to subscribers[^2]. Teams that want zero licensing exposure build their own release or image from source. Audit this before your fleet depends on `docker pull vernemq/vernemq`.

**Netsplits demand a decision.** Cluster state is eventually consistent. During a partition, VerneMQ by default refuses new registrations and subscription changes to avoid divergent state; the `allow_register_during_netsplit` / `allow_subscribe_during_netsplit` family of flags trades consistency for availability[^5]. Decide per deployment — the default manifests as a partial outage during partitions.

**Sessions are node-sticky.** A load balancer spreads connections, but a persistent session's queue lives on its owning node; reconnecting elsewhere migrates the queue. Rolling restarts of large clusters therefore cause reconnect storms and queue-migration churn — drain nodes gradually.

**Disk pressure comes from offline queues.** Persistent sessions with slow or dead consumers accumulate LevelDB-backed offline messages. Cap them (`max_offline_messages`, `max_online_messages`) and use the built-in load shedding, or a fleet of flaky devices will eat your disks.

**Observability is decent:** a Prometheus endpoint and status page on port 8888, `$SYS` topics, Graphite reporting, and `vmq-admin trace` for live per-client protocol tracing — the latter is genuinely useful for debugging misbehaving devices.

**Upgrades:** the 1.x → 2.0 jump (April 2024) modernized the OTP baseline and internals; treat major upgrades as cluster-replacement events rather than in-place rolls, and verify metadata-backend settings match across nodes.

## When to Use / When Not

**Use when:**
- You need a clustered, horizontally scalable MQTT broker with broker-level HA instead of application-level sharding.
- Your auth/integration story maps onto SQL/NoSQL databases or webhooks — the plugin hooks cover this without forking the broker.
- You can build from source (staying purely Apache-2.0) and accept Erlang operational familiarity.
- MQTT 5.0 features (shared subscriptions, message expiry, topic aliases) on a self-hosted broker matter.

**Avoid when:**
- You need one small broker on one box — eclipse/mosquitto is lighter and has no binary-licensing questions.
- You want rich out-of-the-box integrations (Kafka bridges, rule engines, dashboards) and a large community — emqx/emqx is broader and more actively developed.
- You cannot accept EULA-encumbered binaries and also cannot maintain a source build.
- You need strong consistency of subscription state across partitions; the gossip model is availability-biased by design.

## Alternatives

- emqx/emqx — use instead when you want a larger ecosystem: rule engine, data bridges, dashboard, and faster release cadence (Apache-2.0 core, BSL for some enterprise parts).
- eclipse/mosquitto — use instead for single-node or edge deployments where a small C footprint beats clustering.
- hivemq/hivemq-community-edition — use instead if you are Java-shop-native and may grow into HiveMQ's commercial clustering.
- rabbitmq/rabbitmq-server — use instead when MQTT is one protocol among many (AMQP, STOMP) in an existing RabbitMQ estate.
- nanomq/nanomq — use instead for constrained edge gateways where memory footprint dominates.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017-03 | First stable release (project open-sourced 2015)[^1]. |
| 1.6.0 | 2018-10 | 1.x line matures; plugin/integration buildout[^3]. |
| 1.8.0 | 2019-05 | MQTT 5.0 support lands in the 1.x series[^3]. |
| — | 2019 | Binary packages/Docker images move under paid-subscription EULA[^2]. |
| 1.12.0 | 2021-05 | Long-running 1.12.x maintenance line begins[^3]. |
| 2.0.0 | 2024-04 | Major modernization release; new OTP baseline[^3]. |
| 2.1.0 | 2025-04 | Current minor line; 2.1.2 followed 2025-11[^3]. |
| 2.1.3-rc1 | 2026-04 | Latest pre-release; development active[^3]. |

## References

[^1]: VerneMQ documentation. https://docs.vernemq.com
[^2]: VerneMQ commercial support and subscription model. https://vernemq.com/services.html — Docker EULA gate documented at https://docs.vernemq.com/installation/docker
[^3]: GitHub releases, vernemq/vernemq. https://github.com/vernemq/vernemq/releases
[^4]: VerneMQ clustering documentation (metadata plugins, inter-node communication). https://docs.vernemq.com/vernemq-clustering/introduction
[^5]: VerneMQ docs, "Netsplits". https://docs.vernemq.com/vernemq-clustering/netsplits

## Tags

erlang, mqtt, message-broker, pubsub, iot, distributed-systems, clustering, messaging, industrial-iot, self-hosted
