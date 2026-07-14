# osquery/osquery

> Exposes the operating system as a high-performance relational database you query with SQL.

[GitHub repo](https://github.com/osquery/osquery) ·
[Official website](https://osquery.io) ·
[License: Apache-2.0 OR GPL-2.0-only](https://github.com/osquery/osquery/blob/master/LICENSE)

## Overview

osquery represents low-level operating system state — running processes, open sockets, loaded kernel modules, installed packages, users, scheduled tasks, file hashes, hardware events — as SQL tables. You ask questions with `SELECT` statements instead of stitching together `ps`, `netstat`, `lsof`, WMI queries, and platform-specific APIs. The same query text runs unchanged on Linux, macOS, and Windows, which is the project's core value proposition for fleet-wide endpoint visibility.

It was built at Facebook and open-sourced in October 2014[^1]. In 2019 Facebook handed governance to a community-run Technical Steering Committee under the Linux Foundation, and the 4.0 release rewrote the build system from Buck to CMake at the same time[^2]. The typical users are security and detection-engineering teams doing endpoint monitoring, intrusion detection, incident response, and compliance inventory — osquery is instrumentation and a query engine, not a management platform, so it is almost always paired with a separate fleet manager.

The defining tension is that a SQL abstraction over the OS makes some things trivial and hides real costs. A one-line query can accidentally hash every file on disk, walk every process's memory maps, or trigger an expensive WMI provider. osquery answers this with an out-of-process watchdog and per-query resource accounting rather than by making tables cheap, so operating it safely at scale is mostly about knowing which tables are expensive and how often you schedule them.

## Getting Started

Install from the official packages (Homebrew shown; `.deb`/`.rpm`/`.msi` and a macOS `.pkg` are on the downloads page)[^3]:

```bash
brew install --cask osquery
```

Explore interactively with the shell (`osqueryi`), which needs no config or daemon:

```sql
-- processes listening on all interfaces, with the owning binary
SELECT DISTINCT p.name, p.path, lp.port, p.pid
  FROM listening_ports lp
  JOIN processes p USING (pid)
  WHERE lp.address = '0.0.0.0';
```

Run as a monitoring daemon (`osqueryd`) with a scheduled query in `osquery.conf`:

```json
{
  "schedule": {
    "new_root_shells": {
      "query": "SELECT pid, path, cmdline FROM processes WHERE uid = 0;",
      "interval": 300
    }
  }
}
```

Scheduled results are logged as a differential (rows added/removed since the last run) by default, so you get a change stream rather than repeated full snapshots.

## Architecture / How It Works

osquery is a C++ program built on **SQLite's virtual table API**. Each table is a plugin that implements a `Generate` function; SQLite handles parsing, joins, and predicate evaluation, and osquery's tables produce rows on demand. Column constraints from the `WHERE` clause are passed into `Generate` so a table like `file` or `hash` can require a path rather than scanning the world — but only if the query supplies the constraint. Missing that constraint is the most common way to write an accidentally catastrophic query.

Two binaries share one codebase: `osqueryi`, the interactive REPL, and `osqueryd`, the scheduler daemon that also serves logging and distributed-query workflows.

- **Watchdog.** `osqueryd` forks a worker process and the parent monitors its CPU and memory. A query that blows past `--watchdog_level` limits gets its worker killed and the offending query denylisted, protecting the host from a runaway `SELECT`. This out-of-process design is the main reason osquery is considered safe to deploy on production fleets.
- **Evented tables.** Point-in-time tables answer "what is true now." Evented tables (`process_events`, `file_events`, `socket_events`, etc.) use a publisher/subscriber model backed by platform audit facilities — the Linux audit framework or eBPF, Apple's EndpointSecurity, and ETW on Windows — buffering events into a table you then query.
- **Extensions.** Third-party tables, config plugins, and logger plugins can run as separate processes over a Thrift IPC socket. A crashing extension does not take down the daemon.
- **Remote (TLS) plugins.** Config, logger, and "distributed" (ad-hoc query) plugins can point at an HTTPS endpoint. This is the contract every fleet manager implements: the server hands out the config and scheduled queries, receives logs, and pushes live queries.

Platform coverage is uneven by construction: a table exists only where someone implemented it for that OS, so schema availability differs across Linux, macOS, and Windows.

## Production Notes

- **Query cost is yours to manage.** `hash`, `file` with recursive globs, `yara`, `processes` joined against per-process tables, and several WMI-backed Windows tables can be very expensive. Profile with `SELECT * FROM osquery_schedule` and the shell's `.timer`/`--profile` tooling before scheduling anything on a large fleet, and lengthen intervals for heavy queries.
- **The Linux audit socket is single-owner.** osquery's audit-based evented tables and a running `auditd` contend for the same netlink socket; you generally must disable `auditd` or accept that one of them loses events. eBPF-based publishers avoid some of this but need a recent-enough kernel.
- **macOS needs entitlements.** EndpointSecurity-backed event tables require the app to carry Apple's EndpointSecurity entitlement and, in practice, Full Disk Access granted via MDM. Expect this to be the hardest part of a macOS rollout; unmanaged installs silently return empty event tables.
- **Differential vs snapshot logging.** The default differential logging can miss context if you actually wanted the full state each run; `snapshot` queries re-emit everything and cost more storage and bandwidth. Choose deliberately per query.
- **Log volume and cardinality.** Scheduling broad queries across thousands of hosts produces large log streams; the logger plugins (filesystem, syslog, TLS, AWS Kinesis/Firehose) differ in backpressure behavior, and undersized downstream pipelines are a common failure.
- **You still need a fleet manager.** osquery ships no server, UI, or aggregation. Deploying at scale means running one of the managers below, which owns config distribution, log aggregation, and live queries.

## When to Use / When Not

**Use when:**
- You need one query language for endpoint state across Linux, macOS, and Windows.
- You are doing detection engineering, host inventory, compliance evidence, or IR triage and want ad-hoc plus scheduled visibility.
- You want an agent with a resource watchdog and an out-of-process extension model rather than a kernel-resident agent.

**Avoid when:**
- You want turnkey detection with rules, alerting, and a UI out of the box — osquery is the sensor, not the SIEM.
- You need real-time, low-latency threat prevention/blocking; osquery observes and reports, it does not enforce.
- Your target is containers/Kubernetes runtime security specifically — an eBPF syscall-monitoring tool is a closer fit.
- You cannot deploy the required macOS entitlements/MDM and need endpoint events on Apple hardware.

## Alternatives

- fleetdm/fleet — the leading osquery fleet manager; use it when you want a UI, API, and log pipeline around osquery rather than a replacement for it.
- Velocidex/velociraptor — endpoint DFIR and hunting with its own VQL query language; use it when incident response, collection, and live forensics matter more than a stable SQL schema.
- falcosecurity/falco — eBPF/syscall runtime security for containers and Kubernetes; use it when you need real-time detection and cloud-native runtime threat coverage.
- wazuh/wazuh — full open-source HIDS/XDR with rules, alerting, and a dashboard (and an osquery integration); use it when you want batteries-included detection instead of a bare sensor.
- google/grr — remote live forensics and response at scale; use it when large-fleet incident response and remote artifact collection are the priority.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014-10 | Open-sourced by Facebook; SQLite virtual tables over OS state[^1]. |
| 2.0 | 2016 | Windows support work; broader table coverage. |
| 3.0 | 2017 | Continued platform and schema expansion. |
| 4.0 | 2019-08 | Build moved Buck → CMake; governance to Linux Foundation TSC[^2]. |
| 5.0 | 2021-09 | Major version line; ongoing EndpointSecurity/eBPF and Windows ETW work[^4]. |

Minor releases target roughly every two months, tracked via GitHub milestones and per-release checklist issues[^4].

## References

[^1]: Facebook Engineering, "Introducing osquery" — 2014-10-29. https://engineering.fb.com/2014/10/29/security/introducing-osquery/
[^2]: The Linux Foundation, "The Linux Foundation to Host the osquery Community" — 2019-06. https://www.linuxfoundation.org/press/press-release/the-linux-foundation-to-host-the-osquery-community
[^3]: osquery downloads and install instructions. https://osquery.io/downloads/
[^4]: osquery README, "Download & Install" (release cadence and versioning). https://github.com/osquery/osquery/blob/master/README.md

## Tags

c-plus-plus, sql, endpoint-security, osquery, monitoring, intrusion-detection, sqlite, cross-platform, dfir, host-instrumentation
