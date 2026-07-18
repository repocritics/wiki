# processone/ejabberd

> Erlang XMPP server with two decades of production history — now also an MQTT broker, SIP service, and Matrix gateway in one process.

[GitHub repo](https://github.com/processone/ejabberd) ·
[Official website](https://www.process-one.net/ejabberd/) ·
[License: GPL-2.0](https://github.com/processone/ejabberd/blob/master/COPYING)

## Overview

ejabberd is an XMPP (Jabber) server written in Erlang/OTP, started by Alexey Shchepin in 2002 and maintained since 2005 by ProcessOne[^1]. It is one of the longest-lived actively developed open-source servers of any kind: monthly-cadence releases continue in 2026 (26.04 shipped April 2026, last push July 2026), and the protocol surface has grown beyond XMPP to include a built-in MQTT 5 broker (since 19.02)[^2], SIP and STUN/TURN services, and a Matrix federation gateway (since 24.02)[^3]. Its scalability reputation is earned: WhatsApp's original backend was a heavily customized ejabberd fork handling millions of connections per node[^4].

The defining tension is community edition vs. commercial steward. ProcessOne funds development and sells ejabberd Business Edition and Fluux (hosted ejabberd); the GPLv2 community edition is complete and unthrottled, but advanced operational tooling, some scalability modules, and hand-holding live behind the commercial line. The second tension is ecosystem-level: ejabberd is arguably the best implementation of a federated protocol (XMPP) whose consumer mindshare has largely moved to Matrix, Discord, and proprietary chat — which is precisely why the project has been bolting on MQTT, SIP, and Matrix interop rather than remaining XMPP-pure.

At 6.7k stars and 1.5k forks with 215 open issues, activity is steady rather than explosive — typical of mature infrastructure whose users are operators, not framework hobbyists.

## Getting Started

```bash
# Docker (ecs image, the common starting point)
docker run -d --name ejabberd -p 5222:5222 -p 5280:5280 ejabberd/ecs

# create an admin account
docker exec -it ejabberd ejabberdctl register admin localhost passw0rd
```

Minimal `ejabberd.yml` (the server is entirely driven by this YAML file):

```yaml
hosts:
  - example.org

listen:
  -
    port: 5222
    module: ejabberd_c2s
    starttls_required: true
  -
    port: 5280
    module: ejabberd_http
    request_handlers:
      /admin: ejabberd_web_admin

acl:
  admin:
    user: admin@example.org

modules:
  mod_mam: {}          # message archive (XEP-0313)
  mod_muc: {}          # multi-user chat
  mod_offline: {}
  mod_roster: {}
```

Native installers (deb/rpm/run), Homebrew, OS packages, and a Hex package (for embedding in Elixir projects) are also published[^5].

## Architecture / How It Works

ejabberd is a textbook Erlang/OTP application: every client connection, server-to-server connection, and MUC room is a lightweight Erlang process under a supervision tree. A crashed connection process is isolated and restarted without affecting the rest of the node — this, not raw speed, is the core of its reliability story.

- **Routing.** `ejabberd_router` dispatches stanzas by JID; `ejabberd_sm` (session manager) tracks online sessions with pluggable storage (Mnesia, SQL, or Redis).
- **Hooks and modules.** Nearly all functionality is a module implementing the `gen_mod` behaviour, registered per virtual host. Modules attach to named hook points (`ejabberd_hooks`) with priorities — offline storage, MAM archiving, spam filtering all work by intercepting the same hook chain. This is the extension model, and it is genuinely flexible, but hook ordering bugs between third-party modules are a classic failure mode.
- **Storage abstraction.** Mnesia (Erlang's built-in distributed DB) is the default for everything; PostgreSQL/MySQL/SQLite/MS SQL are supported for persistent data, Redis for volatile session state, LDAP for read-only authentication.
- **Performance-critical deps as NIFs.** XML parsing (`fast_xml`), TLS (`fast_tls`, wrapping OpenSSL), and stringprep are C NIFs maintained by ProcessOne as separate libraries. They make the numbers possible, but a NIF segfault takes down the whole VM — the one hole in the OTP isolation story.
- **Clustering.** Nodes form a full mesh over Erlang distribution, sharing routing and session tables via Mnesia replication. Any node accepts any user; stanzas are routed to the node holding the session[^6].
- **MQTT/SIP/Matrix** reuse the same listener, auth, ACL, and clustering infrastructure rather than being separate daemons — one config, one cluster, multiple protocols.

## Production Notes

**Mnesia is the default and the main footgun.** It works well for a single node or small cluster, but: it has size limits on disc tables, cluster network partitions cause split-brain that requires manual reconciliation, and schema changes across major upgrades can be awkward. The well-trodden path for serious deployments is PostgreSQL for persistent data (rosters, MAM, offline) while keeping only volatile tables in Mnesia — the docs and `ejabberdctl` provide export tooling, but plan the migration before you have data you care about.

**Erlang distribution is not WAN-friendly.** Clustering assumes a trusted low-latency network, a shared secret cookie, and (by default) unencrypted inter-node traffic on dynamically allocated ports. Cross-datacenter clustering requires TLS distribution setup and careful firewalling; most operators instead run independent clusters per region and federate over s2s.

**Upgrades need release notes reading.** The YY.MM cadence is roughly quarterly and individual upgrades are usually smooth, but SQL schema changes ship periodically and must be applied manually (there are old and "new" SQL schemas — new deployments should start on the new one). Skipping many versions compounds this.

**Configuration surface is enormous.** Hundreds of options across dozens of modules, all in one YAML file. Validation at startup is good (unknown options fail loudly), but semantic misconfiguration — shaper/ACL interactions, module option interplay — is where operator time goes. `ejabberdctl` and the web admin cover day-2 operations; a REST API (`mod_http_api` + `ejabberd_oauth`) exists for automation.

**Capacity.** A single tuned node handles hundreds of thousands of concurrent connections; kernel tuning (file descriptors, TCP buffers) and Erlang VM flags (`+P`, ports) are the levers. The famous multi-million-per-node WhatsApp numbers involved heavy FreeBSD and codebase customization[^4] — treat them as an existence proof, not a default.

**Certificates.** Built-in ACME (Let's Encrypt) support and SNI make TLS management notably less painful than in most self-hosted servers.

**Community modules** live in the separate `ejabberd-contrib` repo with varying maintenance quality — audit before deploying.

## When to Use / When Not

**Use when:**
- You need a self-hosted, standards-based (XMPP/XEP) messaging backbone you fully control — federation, e2e via OMEMO clients, no vendor lock.
- You are building high-fanout realtime infrastructure (IoT via MQTT, presence, pubsub) and want carrier-grade connection density per node.
- You want one clustered process speaking XMPP + MQTT + SIP instead of three daemons.
- Long-horizon maintenance matters: 20+ years of history and a commercial steward is rare insurance.

**Avoid when:**
- You are adding chat to a product and would be better served by a chat API/SDK (Sendbird, Stream) or a simple WebSocket layer — XMPP's XML stanza model and XEP sprawl carry real complexity tax.
- Your team has no Erlang exposure and will need to debug beyond configuration — the hook system and Mnesia internals assume OTP literacy.
- You want the Matrix ecosystem proper (Element clients, Matrix bridges); ejabberd's Matrix gateway is interop, not a Matrix homeserver.
- You need rich moderation/communities UX out of the box — that is client- and integration-work on XMPP.

## Alternatives

- bjc/prosody — Lua XMPP server; far lighter and easier to configure, better for small/personal servers, less suited to large clusters.
- esl/MongooseIM — Erlang Solutions' ejabberd fork; use when you want a more product-engineering-oriented XMPP platform with its own REST/GraphQL APIs.
- igniterealtime/Openfire — Java XMPP server with a friendly admin UI; use in Java shops that value ease of administration over connection density.
- element-hq/synapse — reference Matrix homeserver; use when you want the Matrix ecosystem and clients rather than XMPP.
- emqx/emqx — use when you only need MQTT at scale and none of the XMPP/SIP surface.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2002-11 | First code by Alexey Shchepin[^1]. |
| 2.0.0 | 2008-02 | Major rewrite of the 1.x line. |
| 2.1.13 | 2013 | Final 2.x release; 3.0 alphas abandoned in favor of merging work into mainline. |
| 13.03 | 2013-03 | Switch to date-based YY.MM versioning; YAML config arrived later that year (13.10). |
| 15.02 | 2015-02 | Elixir integration; ejabberd published on Hex. |
| 19.02 | 2019-02 | Built-in MQTT broker (`mod_mqtt`)[^2]. |
| 24.02 | 2024-02 | Matrix federation gateway (`mod_matrix_gw`)[^3]. |
| 25.10 | 2025-10 | Part of the ongoing ~quarterly cadence. |
| 26.04 | 2026-04-20 | Latest release at time of writing[^7]. |

## References

[^1]: ejabberd — Wikipedia (project origins, initial 2002 release). https://en.wikipedia.org/wiki/Ejabberd
[^2]: ejabberd 19.02 release (MQTT support). https://github.com/processone/ejabberd/releases/tag/19.02
[^3]: ejabberd 24.02 release (Matrix gateway). https://github.com/processone/ejabberd/releases/tag/24.02
[^4]: "The WhatsApp Architecture Facebook Bought For $19 Billion" — High Scalability, 2014-02-26. http://highscalability.com/blog/2014/2/26/the-whatsapp-architecture-facebook-bought-for-19-billion.html
[^5]: ejabberd Docs — Installation. https://docs.ejabberd.im/admin/install/
[^6]: ejabberd Docs — Clustering. https://docs.ejabberd.im/admin/guide/clustering/
[^7]: ejabberd releases. https://github.com/processone/ejabberd/releases

## Tags

erlang, xmpp, mqtt, sip, messaging, chat-server, realtime, pubsub, federation, self-hosted, iot
