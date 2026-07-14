# crowdsecurity/crowdsec

> Crowdsourced, log-driven intrusion detection and remediation — Fail2ban's ideas rebuilt around a shared threat-intelligence network.

[GitHub repo](https://github.com/crowdsecurity/crowdsec) ·
[Official website](https://crowdsec.net) ·
[License: MIT](https://github.com/crowdsecurity/crowdsec/blob/master/LICENSE)

## Overview

CrowdSec is a Go security engine that reads logs (and, more recently, live HTTP traffic), detects malicious behavior via declarative scenarios, and turns detections into *decisions* that separate agents enforce. It is frequently described as "a modern Fail2ban," and that framing is accurate for the single-host case: both watch logs and ban IPs. The differences are the point of the project — CrowdSec decouples detection from remediation, ships a hub of community-maintained parsers and scenarios, and optionally shares anonymized attack signals with a central network in exchange for a curated community blocklist[^1].

The defining tension is participation vs. privacy and self-containment. The headline value — the community blocklist, populated by every consenting deployment worldwide — only exists if you send signals (offending IP, scenario, timestamp) to CrowdSec's Central API. You can run fully offline, but then CrowdSec is "Fail2ban with a nicer architecture and a YAML hub," and you forgo the reason most people adopt it. Organizations with strict egress or data-sharing policies have to weigh that trade explicitly.

The second recurring surprise is that CrowdSec detects but does not block on its own. Enforcement lives in a separate process — a *bouncer* / *remediation component* (firewall, nginx, Cloudflare, Traefik, etc.), installed and configured independently. First-time users often stand up the engine, watch it generate alerts, and wonder why nothing is being banned.

## Getting Started

```bash
# Debian/Ubuntu — repo + engine
curl -s https://install.crowdsec.net | sudo sh
sudo apt install crowdsec

# Detection is live now — but nothing is enforced until you add a bouncer:
sudo apt install crowdsec-firewall-bouncer-iptables
```

Inspect and manage via `cscli`:

```bash
cscli metrics                 # acquisition + parse + bucket throughput
cscli decisions list          # currently active bans (local + community)
cscli alerts list             # what triggered them
cscli hub list                # installed parsers / scenarios / collections
cscli collections install crowdsecurity/nginx   # add nginx detection
cscli decisions add --ip 1.2.3.4 --duration 4h  # manual ban
```

## Architecture / How It Works

The pipeline inside a single Security Engine runs in stages:

1. **Acquisition** — tail files, journald, Docker/Kubernetes streams, or listen for syslog/AppSec traffic. Configured in `acquis.yaml`.
2. **Parsing** — grok-based parsers normalize raw lines into structured events. Parsers are ordered stages and can enrich (geoIP, reverse DNS, whitelisting).
3. **Scenarios** — the detection unit, modeled as **leaky buckets**. Events pour into a bucket keyed by (say) source IP; if it overflows within a time window (N failed logins in M seconds), the bucket emits an alert. Scenario logic and parser expressions use the `expr` expression language, not regex-only rules.
4. **Local API (LAPI)** — alerts become *decisions* (ban IP X until T) stored in a database. LAPI is the hub every other component talks to.
5. **Remediation** — bouncers poll LAPI and enforce decisions in their own domain (iptables/nftables, nginx Lua, a CDN API). They never see logs; they only consume decisions.

Above the local node sits the **Central API (CAPI)**: consenting engines push signals up and pull the community blocklist down, so an IP that attacked someone else yesterday is blocked on your host today. The **Hub** (hub.crowdsec.net, MIT-licensed) distributes parsers, scenarios, and *collections* (bundles for nginx, sshd, WordPress, etc.). Data is persisted through the `ent` ORM — SQLite by default, MySQL/Postgres for multi-node LAPI setups.

The 1.0 rewrite (2021) introduced LAPI and formally split the agent from bouncers[^2]; this decoupling is the whole architecture and the source of most operational confusion. More recently CrowdSec grew an **AppSec Component** — an inline WAF that inspects live HTTP requests (supporting SecLang / ModSecurity-style rules via a Coraza-derived engine) rather than only post-hoc logs[^3]. This moves CrowdSec from pure IDS toward inline IPS/WAF territory and is comparatively young.

## Production Notes

- **The bouncer is the deployment.** The engine alone changes nothing. Choose the bouncer that matches your enforcement point (host firewall vs. reverse proxy vs. CDN); an IP banned at nginx still reaches the box for non-HTTP services.
- **Central API opt-in is a real decision, not a checkbox.** Enrolling shares attack metadata externally. Air-gapped or compliance-bound deployments run offline and lose the community blocklist; document that trade in your threat model.
- **False positives are cheap to cause and load-bearing.** Aggressive scenarios plus a shared blocklist means a misconfigured whitelist can ban your own CI, health-checkers, or NAT'd office. Maintain `cscli` whitelist parsers for known-good ranges before turning enforcement on.
- **Decision TTLs and DB growth.** Every decision has a duration; the DB (SQLite by default) accumulates alerts and decisions. High-traffic edge nodes should watch DB size and use a real database for shared/HA LAPI rather than the default file.
- **AppSec/WAF is newer and inline.** Running detection in the request path adds latency and a failure mode the log-based path never had (a crashed/blocking AppSec component can affect live traffic). Treat it as less battle-tested than the log-processing core.
- **The Console is a SaaS upsell.** The open-source engine is genuinely complete on its own, but visualization, multi-instance management, and premium/paid blocklists live behind app.crowdsec.net. Nothing forces you there, but the polished "single pane of glass" experience is the commercial layer.
- **Version skew across components.** Engine, LAPI, bouncers, and hub items version independently. Upgrades occasionally require bumping bouncers or re-pulling hub content; `cscli hub upgrade` and matching bouncer packages are part of routine maintenance.

## When to Use / When Not

**Use when:**
- You run internet-facing Linux hosts (ssh, nginx, web apps) and want brute-force / scan / bot protection that improves as the network sees more attacks.
- You want detection and enforcement decoupled — detect on a log aggregator, remediate at the CDN or firewall ("detect here, remedy there").
- You'd benefit from a maintained hub of scenarios instead of hand-writing regex jails.

**Avoid when:**
- You cannot or will not share signals externally *and* your only reason to adopt is the community blocklist — offline mode narrows the value sharply.
- You need a single self-contained banning daemon with no extra moving parts on one box — Fail2ban is simpler.
- You need a mature, high-assurance inline WAF today — dedicated WAF/IDS engines (Coraza, ModSecurity, Suricata) are more established for that specific job.

## Alternatives

- fail2ban/fail2ban — single-host, regex + iptables, no sharing; simpler when one box is all you have.
- corazawaf/coraza — embeddable Go WAF (SecLang/OWASP CRS) when you want an inline WAF library rather than a log-driven engine.
- owasp-modsecurity/ModSecurity — the reference WAF engine + OWASP Core Rule Set for HTTP-layer filtering.
- OISF/suricata — network IDS/IPS at the packet layer; complementary rather than overlapping.
- wazuh/wazuh — full host-based IDS/SIEM/XDR when you want centralized security monitoring, not just IP remediation.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-05 | Repository opened; first public 0.x releases[^1]. |
| 1.0 | 2021 | Local API introduced; agent and bouncers decoupled[^2]. |
| 1.4 | 2022 | Hub/collection maturation, multi-source acquisition improvements. |
| 1.5 | 2023 | AppSec (WAF) component introduced, initially experimental[^3]. |
| 1.6 | 2024 | AppSec and engine hardening; ongoing bouncer ecosystem growth. |

## References

[^1]: CrowdSec README and project site. https://github.com/crowdsecurity/crowdsec — https://crowdsec.net
[^2]: CrowdSec documentation, "Local API (LAPI)" and component architecture. https://doc.crowdsec.net/
[^3]: CrowdSec documentation, "Application Security Component (AppSec / WAF)." https://doc.crowdsec.net/docs/next/appsec/intro

## Tags

go, security, ids, ips, waf, intrusion-detection, threat-intelligence, log-analysis, fail2ban-alternative, self-hosted, linux, crowdsourced
