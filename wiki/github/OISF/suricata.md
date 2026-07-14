# OISF/suricata

> Multi-threaded network IDS, IPS, and network security monitoring engine that speaks a Snort-compatible rule language.

[GitHub repo](https://github.com/OISF/suricata) ·
[Official website](https://suricata.io) ·
[License: GPL-2.0](https://github.com/OISF/suricata/blob/main/LICENSE)

## Overview

Suricata is a network threat-detection engine developed by the Open Information Security Foundation (OISF), first released as a stable 1.0 in 2010 as a multi-threaded answer to the then single-threaded Snort[^1]. It inspects live or recorded network traffic, matches it against signatures, parses application-layer protocols, extracts files, and emits structured events. It runs in three overlapping roles: passive IDS (alert only), inline IPS (drop/reject packets), and NSM (log protocol metadata and flow records regardless of alerts).

Its defining design decision is a threaded, flow-pinned packet pipeline that scales across CPU cores by hashing flows onto worker threads. This is what lets a single box push detection at multi-gigabit rates, but it is also the source of Suricata's central tension: raw capability versus tuning burden. Out of the box on a busy link it will drop packets. Reaching line rate demands deliberate work on NIC RSS queues, CPU affinity, disabled hardware offload, memcap sizing, and pattern-matcher selection. Suricata rewards operators who tune it and punishes those who deploy it like an appliance.

The second tension is inspection versus encryption. Suricata reads a Snort-style rule syntax (Emerging Threats and Talos ship large rulesets for it[^5]), but the payload those rules inspect is increasingly TLS-encrypted. On modern traffic much of Suricata's value shifts from deep-packet content matching to metadata: TLS SNI/JA3 fingerprints, DNS, HTTP headers before upgrade, and flow records rendered as EVE JSON[^3].

## Getting Started

```bash
# Debian/Ubuntu — OISF maintains current builds in a PPA
sudo add-apt-repository ppa:oisf/suricata-stable
sudo apt update && sudo apt install suricata

# Fetch and manage rulesets (ET Open by default)
sudo suricata-update

# IDS mode on a live interface
sudo suricata -i eth0 -c /etc/suricata/suricata.yaml

# Offline: replay a capture, write logs to ./out
suricata -r traffic.pcap -l ./out/
```

A minimal rule (Suricata/Snort signature syntax):

```
alert http any any -> any any ( \
    msg:"Suspicious User-Agent"; flow:established,to_server; \
    http.user_agent; content:"evil-bot"; nocase; \
    sid:1000001; rev:1;)
```

Alerts, flow records, and protocol logs land in `eve.json` — one JSON object per line, the format most SIEM/ELK pipelines ingest[^3].

## Architecture / How It Works

The packet path is a staged pipeline[^2]:

1. **Capture** — packets enter through a capture module: AF_PACKET or Netmap on Linux, PF_RING, DPDK for kernel-bypass, NFQUEUE for inline IPS, or `libpcap` for offline files.
2. **Decode** — link/network/transport headers are parsed; flows are tracked in a hash table with configurable memcaps.
3. **Stream reassembly** — TCP segments are reordered and reassembled so detection sees a coherent byte stream, not raw packets.
4. **App-layer parsing** — protocol parsers (HTTP, TLS, DNS, SMB, SSH, SMTP, DHCP, and many more) produce typed fields and transactions. Many of these parsers were rewritten in Rust starting with 4.0 (2017); new protocol support is Rust-first[^1].
5. **Detection** — a multi-pattern matcher (MPM) does a fast first-pass content prefilter, then surviving rules are fully evaluated. Intel Hyperscan is the recommended MPM backend for throughput[^6].
6. **Output** — EVE JSON, fast.log, PCAP logging, file extraction, stats, and Unix-socket control.

Threading is governed by **runmodes**. `workers` mode pins a full pipeline per thread and relies on the NIC's RSS to load-balance flows across queues; `autofp` decouples capture from detection with flow-based hand-off. Flow pinning is what keeps a TCP conversation on one core so reassembly state stays local.

The engine is C at its core with a growing Rust surface (protocol parsers, some detection logic) linked in. Configuration is one large `suricata.yaml`; there is no separate control plane — the running behavior is the file plus the loaded rules plus the capture method, all coupled.

## Production Notes

**Hardware offload must be disabled.** GRO/LRO/GSO on the NIC coalesce segments before Suricata sees them and corrupt stream reassembly. `ethtool -K eth0 gro off lro off gso off tso off` is a required step, not an optimization.

**Packet loss is silent unless you watch for it.** `capture.kernel_drops` in `stats.log` (or `suricatasc -c dump-counters`) is the number that matters. Rising drops mean missed detection. Fixing it means more RSS queues, CPU pinning via `threading.cpu-affinity`, and Hyperscan — not a bigger ruleset.

**Memcaps degrade quietly.** Flow, stream, reassembly, and defrag each have a memcap in the YAML. Hitting one doesn't crash; it drops the oldest state, so detection quietly gets worse under load. Size them against real traffic and monitor the `memcap_pressure` counters.

**IPS mode raises the stakes.** Inline via NFQUEUE or AF_PACKET IPS bridges two interfaces; a crash, stall, or memcap stall can take the network segment offline (the README says as much). Fail-open vs fail-closed behavior must be decided deliberately. Run IPS only after validating the config in IDS mode.

**Ruleset bloat is a real cost.** ET Open plus other feeds is tens of thousands of rules; loading and evaluating them costs memory and CPU. Use `suricata-update` to enable only relevant categories, and profile with rule-profiling builds. Flow bypass (kernel or engine) sheds large "elephant" flows to keep up with line rate.

**EVE JSON volume.** Enabling all event types (especially `flow` and `anomaly`) on a busy link can produce enormous log volume. Tune which event types and which fields are emitted before pointing it at a SIEM.

**Encrypted traffic is a blind spot.** With TLS 1.3, payload inspection is gone; you get SNI, certificate metadata, JA3/JA3S, and timing. Detection strategy must account for this rather than assuming content rules fire.

**Upgrades.** Major versions change `suricata.yaml` defaults and occasionally rule keyword behavior. Read the upgrade doc between majors; diff your config against the shipped example. 6.0 and 7.0 are the long-term-support lines[^7].

## When to Use / When Not

**Use when:**
- You need signature-based detection with a Snort-compatible ecosystem (ET, Talos) but want multi-core throughput.
- You want inline IPS enforcement, not just alerting.
- You want rich protocol/flow telemetry (EVE JSON) feeding a SIEM, with or without alerts.
- You can dedicate effort to NIC/CPU/memcap tuning and ongoing rule management.

**Avoid when:**
- You want behavioral/protocol analysis and scripting rather than signatures — Zeek fits that better.
- You need full packet capture and retrospective search — Suricata detects and logs metadata; it is not a PCAP index.
- You have no capacity to tune and monitor drops; an unmaintained sensor gives false assurance.
- Your traffic is almost entirely encrypted and you have no metadata-based detection strategy.

## Alternatives

- snort3/snort3 — the incumbent signature engine, multi-threaded since v3; use Snort when you're standardized on Cisco/Talos tooling or need feature parity with existing Snort deployments.
- zeek/zeek — network security monitoring via protocol analysis and an event-scripting language; use Zeek when you want programmable, behavior-oriented detection rather than pattern signatures (the two are frequently run side by side).
- arkime/arkime — large-scale full packet capture and session indexing; use Arkime when the goal is retrospective search over stored traffic, not real-time signature detection.
- ntop/nDPI — a deep-packet-inspection library for protocol classification; use nDPI when you need embeddable protocol identification without a full detection engine.
- wazuh/wazuh — host-based detection and SIEM; use Wazuh when the threat surface is endpoints/logs rather than the wire (often paired with Suricata as its network sensor).

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2010-07 | First stable release; multi-threaded engine[^1]. |
| 2.0 | 2014-03 | App-layer improvements, EVE output groundwork. |
| 3.0 | 2016-01 | Netmap/PF_RING maturation, protocol coverage. |
| 4.0 | 2017-09 | Rust protocol parsers introduced[^1]. |
| 5.0 | 2019-10 | JA3, expanded app-layer, dataset keyword. |
| 6.0 | 2020-10 | LTS line; performance and protocol additions. |
| 7.0 | 2023-07 | LTS; more Rust parsers, DPDK, conditional logging[^7]. |
| 8.0 | 2025 | Latest major line; continued Rust migration. |

## References

[^1]: Suricata / OISF project overview and history. https://suricata.io and https://github.com/OISF/suricata
[^2]: Suricata User Guide (architecture, runmodes, packet path). https://docs.suricata.io
[^3]: EVE JSON output format. https://docs.suricata.io/en/latest/output/eve/eve-json-output.html
[^4]: suricata-update rule management tool. https://docs.suricata.io/en/latest/rule-management/suricata-update.html
[^5]: Emerging Threats Open ruleset. https://rules.emergingthreats.net/
[^6]: Intel Hyperscan multi-pattern matching library. https://github.com/intel/hyperscan
[^7]: Suricata 7.0 release announcement. https://forum.suricata.io/ and https://suricata.io/blog/

## Tags

c, network-security, ids, ips, nsm, intrusion-detection, packet-inspection, threat-detection, cybersecurity, eve-json, rust
