# eclipse-mosquitto/mosquitto

> The default MQTT broker of the small-device world — a single-process C
> implementation of MQTT 5.0/3.1.1 that runs anywhere from a home router to a rack.

[GitHub repo](https://github.com/eclipse-mosquitto/mosquitto) ·
[Official website](https://mosquitto.org) ·
[License: EPL-2.0 OR BSD-3-Clause](https://github.com/eclipse-mosquitto/mosquitto/blob/master/LICENSE.txt)

## Overview

Mosquitto is an open-source MQTT broker plus a C/C++ client library
(`libmosquitto`) and CLI tools (`mosquitto_pub`, `mosquitto_sub`,
`mosquitto_rr`, `mosquitto_passwd`, `mosquitto_ctrl`). It implements MQTT 5.0,
3.1.1, and 3.1[^1]. Written by Roger Light starting in 2009, it moved to the
Eclipse Foundation's IoT working group in 2014[^2] and is dual-licensed
EPL-2.0 / EDL-1.0 (BSD-3-Clause). Development is sponsored by Cedalo, which
employs the maintainer and sells a commercial management layer on top[^3].

Its GitHub numbers (~11k stars, ~2.6k forks) understate its footprint:
Mosquitto ships in every major Linux distro, is the de facto broker for Home
Assistant and most hobbyist/industrial IoT setups, and its Docker image is one
of the most-pulled infrastructure images on Docker Hub. The repo is actively
maintained (pushed within weeks as of mid-2026), though overwhelmingly by a
single author — a real bus-factor consideration for critical infrastructure.

The defining tradeoff: Mosquitto is a **single-process, essentially
single-threaded broker with no clustering**. That is why it fits in a few
hundred kilobytes and runs on OpenWrt routers and Raspberry Pis — and also why
it has a hard vertical-scaling ceiling. The right answer for one box; the
wrong starting point for a horizontally scaled fleet of millions of clients.

## Getting Started

```bash
sudo apt install mosquitto mosquitto-clients   # Debian/Ubuntu; brew on macOS
# Docker (2.x image listens on localhost only without a config file)
docker run -it -p 1883:1883 eclipse-mosquitto:2
```

Since 2.0, remote access requires an explicit listener and authentication[^4]:

```conf
# /etc/mosquitto/conf.d/local.conf
listener 1883
allow_anonymous false
password_file /etc/mosquitto/passwd
persistence true
persistence_location /var/lib/mosquitto/
```

```bash
sudo mosquitto_passwd -c /etc/mosquitto/passwd alice
mosquitto_sub -h localhost -u alice -P 'secret' -t 'sensors/#' -v
mosquitto_pub -h localhost -u alice -P 'secret' -t 'sensors/temp' -m '21.5'
```

## Architecture / How It Works

The broker is a classic C event loop (epoll on Linux, poll/select elsewhere).
All client sessions, the subscription tree, and retained messages live in
process memory. Topic matching walks a hierarchy keyed on the `/` levels of
the topic, so wildcard subscriptions (`+`, `#`) resolve by tree traversal, not
per-message scans. There are no worker threads for client traffic: parsing,
routing, TLS, and ACL checks all share one core.

**Persistence** is snapshot-based, not write-ahead: the in-memory state is
serialized to `mosquitto.db` every `autosave_interval` seconds (default 1800).
A crash loses everything since the last snapshot. Version 2.1 adds an optional
SQLite-backed persistence plugin as a more durable alternative[^5].

**Bridges**, not clusters, are the multi-broker story: a broker opens an
outbound MQTT connection to another broker and forwards topics with optional
remapping. This gives edge-to-cloud federation but no shared session state, no
failover, and loop risk if topologies are drawn carelessly.

**Security** is layered: TLS via OpenSSL (including client certificates),
password files, ACL files, and a C plugin API for auth/authz callbacks. The
Dynamic Security plugin (bundled since 2.0) moves clients/groups/roles into a
JSON store administered at runtime via `mosquitto_ctrl`[^4]; third-party
database-auth plugins build on the same interface. 2.1 adds an optional broker
HTTP API via libmicrohttpd[^1].

## Production Notes

- **The 1.6 → 2.0 upgrade is the classic footgun.** 2.0 removed the implicit
  open listener: without explicit `listener` and auth config the broker binds
  localhost only, and `allow_anonymous` defaults to false[^6]. Same for the
  `eclipse-mosquitto:2` Docker image — "broker runs but nothing can connect"
  is the most common issue-tracker complaint of the 2.x era.
- **Message loss by design.** `max_queued_messages` (default 1000) silently
  drops QoS 1/2 messages queued for offline persistent sessions. Combined with
  snapshot persistence, this is not a durable queue — if you need one, you
  want a message queue, not MQTT.
- **Vertical ceiling.** One process, one core for the main loop. Tens of
  thousands of mostly-idle clients on tuned Linux (raise `LimitNOFILE`) is
  realistic; high fan-out rates or heavy TLS churn saturate far earlier. TLS
  reconnect storms after a restart are the canonical incident — stagger client
  backoff.
- **No HA story.** Failover is DIY: active/passive with a VIP and copied
  persistence file, or bridge topologies. Cross-node session takeover does not
  exist.
- **Monitoring** is via `$SYS/#` topics; Prometheus export needs a third-party
  exporter. WebSockets listeners work but are less battle-tested at high
  connection counts than raw TCP.

## When to Use / When Not

**Use when:**
- You need an MQTT broker on constrained hardware — edge gateways, routers,
  SBCs, industrial boxes.
- Single-node scale (thousands to low tens of thousands of clients) is enough.
- You want the ecosystem default: every MQTT client library, tutorial, and
  Home Assistant integration is tested against it first.
- You need embeddable client-side MQTT in C via `libmosquitto`.

**Avoid when:**
- You need horizontal scale-out or broker-level HA — clustering does not exist
  here at any price.
- You need durable, exactly-once queue semantics; QoS 2 plus snapshot
  persistence is not a Kafka substitute.
- Your team wants shared-nothing clusters and dashboards out of the box.

## Alternatives

- emqx/emqx — use instead when you need clustered scale-out to millions of
  connections with a management dashboard.
- vernemq/vernemq — Erlang/OTP broker; use when you want masterless clustering
  and are comfortable operating BEAM systems.
- hivemq/hivemq-community-edition — Java broker with a strong extension SDK;
  the commercial edition adds clustering.
- nanomq/nanomq — use at the edge when you want Mosquitto-like footprint but
  multi-core throughput.
- mochi-mqtt/server — use when you want to embed a broker inside a Go
  application rather than run a separate daemon.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2009 | Initial releases by Roger Light[^2]. |
| 1.0 | 2012 | First stable release. |
| 1.4 | 2015-02 | First release as an Eclipse project; WebSockets support. |
| 1.5 | 2018-05 | Per-listener settings, performance work. |
| 1.6 | 2019-04 | MQTT 5.0 support. |
| 2.0 | 2020-12 | Breaking security defaults (explicit listeners, anonymous off), Dynamic Security plugin, privilege dropping[^6]. |
| 2.1 | 2026-01 | SQLite persistence option, broker HTTP API, new admin utilities[^5]. |

## References

[^1]: Eclipse Mosquitto README and homepage. https://mosquitto.org/
[^2]: Eclipse IoT project page, "Eclipse Mosquitto". https://projects.eclipse.org/projects/iot.mosquitto
[^3]: Cedalo GmbH — sponsors Mosquitto development. https://cedalo.com/
[^4]: `mosquitto.conf` man page and Dynamic Security documentation. https://mosquitto.org/man/mosquitto-conf-5.html
[^5]: Mosquitto ChangeLog (v2.1.x). https://github.com/eclipse-mosquitto/mosquitto/blob/master/ChangeLog.txt
[^6]: Roger Light, "Version 2.0.0 released" — 2020-12-03. https://mosquitto.org/blog/2020/12/version-2-0-0-released/

## Tags

c, mqtt, message-broker, iot, pubsub, embedded, eclipse-iot, networking, edge-computing, protocol-server
