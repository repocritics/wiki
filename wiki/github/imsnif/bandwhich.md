# imsnif/bandwhich

> A terminal utility that shows current network bandwidth broken down by process, connection, and remote host.

[GitHub repo](https://github.com/imsnif/bandwhich) ·
[License: MIT](https://github.com/imsnif/bandwhich/blob/main/LICENSE)

## Overview

bandwhich is a Rust CLI that answers one question interactively: *what is using my
network right now, and where is it going?* It renders a live terminal dashboard with
three tables — processes, connections, and remote addresses — each showing up and down
throughput. The niche it fills is per-process attribution: tools like `iftop` show you
which remote hosts are busy but not which local program is talking to them, and
interface-level tools (`nload`, `bmon`) show only aggregate throughput. bandwhich ties a
packet back to the owning PID, which is the piece operators usually actually want[^1].

The project was created in 2019 by Aram Drevekenin (imsnif, also the author of the Zellij
terminal multiplexer) and reached wide visibility early via Hacker News and the Rust CLI
community. As of 2026 it sits at roughly 11.9k stars with a MIT license[^2]. Its defining
present-day tension is stated bluntly in its own README: the project is in **passive
maintenance**. Critical bugs get fixed, but no new features are planned, explicitly for
lack of funding and maintainer time rather than because the tool is considered finished —
the maintainers actively solicit co-maintainers[^3]. Treat it as a stable, feature-frozen
utility, not an evolving product.

The second inherent tension is privilege. Packet sniffing and mapping sockets to processes
both require elevated capabilities, so bandwhich cannot be a drop-in unprivileged tool the
way `htop` is; every install ends with a privilege-granting step that varies by OS.

## Getting Started

bandwhich is packaged widely (see the Repology matrix in the README). Common installs:

```sh
# From crates.io
cargo install bandwhich

# Debian/Ubuntu (if packaged), Arch, Homebrew, etc.
brew install bandwhich
```

On Linux it needs elevated privileges to capture packets and read `/proc`. Grant
capabilities to the binary once (preferred on single-user or all-trusted machines):

```sh
sudo setcap cap_sys_ptrace,cap_dac_read_search,cap_net_raw,cap_net_admin+ep \
  $(command -v bandwhich)
bandwhich                      # now runs unprivileged
```

Or escalate per-run: `sudo bandwhich` (note `sudo` may drop your `$PATH`; use
`sudo env "PATH=$PATH" bandwhich` if you hit "command not found"). On Windows you must
install [npcap](https://npcap.com/) first. Useful flags: `-i eth0` to pin an interface,
`-r` for machine-readable raw output, `-n` to skip reverse-DNS, `-p`/`-c`/`-a` to show a
single table, `-u si-bytes` to change units.

## Architecture / How It Works

bandwhich does two independent jobs and joins their results in the TUI:

1. **Packet capture.** It sniffs a network interface and records the size and direction of
   each IP packet. This is where `cap_net_raw`/`cap_net_admin` (Linux) or npcap (Windows)
   are needed. Throughput is derived by summing packet sizes per socket over each refresh
   interval — it is measuring what crosses the interface, not what the kernel's own
   counters report, so numbers are approximate and interface-scoped.
2. **Socket-to-process mapping.** To attribute a connection to a program it must resolve
   `(local addr, remote addr, protocol) → PID`. It does this with a per-OS backend: the
   `/proc` filesystem on Linux (hence `cap_sys_ptrace`/`cap_dac_read_search` to read other
   users' `/proc/<pid>/fd`), `lsof` on macOS, and the Windows API on Windows[^1].

Reverse DNS runs asynchronously and best-effort: remote IPs are resolved to hostnames in
the background so the display never blocks on a slow resolver. The UI is responsive to
terminal size and will drop columns or whole tables when there is no room. Because the two
backends differ per platform, feature parity and accuracy differ per platform too — the
Linux `/proc` path is the most complete; macOS depends on shelling out to `lsof`, which is
slower and coarser.

There is no daemon, no persistence, and no historical store: bandwhich is a point-in-time
observer. `-t` shows cumulative totals for the session, but nothing is written to disk
unless you enable `--log-to` for debug logging.

## Production Notes

- **It is not a monitoring system.** There is no time-series storage, no alerting, no
  export to Prometheus/InfluxDB, and no headless mode beyond `-r` raw stdout. For dashboards
  or historical graphs you want ntopng, vnstat, or an eBPF exporter — not bandwhich.
- **Numbers are interface-measured, not authoritative.** Counts come from sniffed packet
  sizes, so they can diverge from `ip -s link` kernel counters, will miss offloaded/GRO
  effects, and only reflect the interface(s) being watched. Good for "who is hogging the
  link," not for billing-grade accounting.
- **The privilege step is the top support issue.** `setcap` must be re-applied after every
  reinstall/upgrade because it targets the specific binary inode. Package upgrades silently
  drop the capabilities, and the tool then fails or shows empty tables. On hardened systems
  `setcap` may be disallowed on a `$HOME`-installed binary, forcing `sudo`.
- **macOS is the weaker platform.** Relying on `lsof` for socket mapping means higher
  overhead and occasional gaps versus Linux's `/proc`. On busy hosts the mapping can lag the
  capture.
- **Passive maintenance is a real planning input.** Since the project is feature-frozen and
  under-resourced[^3], do not adopt it expecting upstream to add a feature you need. Pin the
  version you validated; regressions are unlikely but fixes may be slow. If you depend on it
  operationally, the maintainers' open invitation is to become a co-maintainer.
- **Container/namespace caveats.** Inside containers, `/proc` PID namespacing means bandwhich
  sees the processes of the namespace it runs in; running it on the host to observe a
  container's traffic will attribute packets to the container runtime, not the in-container
  process.

## When to Use / When Not

**Use when:**
- You need to know *which process* is saturating a link, interactively, right now.
- You want a single self-contained binary with no agent, config, or database.
- You're on Linux where the `/proc` backend gives the fullest, fastest picture.

**Avoid when:**
- You need historical graphs, alerting, or metrics export (use a real monitoring stack).
- You need billing-grade or kernel-authoritative byte counts.
- You can't grant capture privileges, or you're on a locked-down host where `setcap`/`sudo`
  is unavailable.
- You need long-term upstream feature development — the project is explicitly frozen.

## Alternatives

- raboof/nethogs — per-process bandwidth like bandwhich but a mature C++ tool; use when you
  want the same "by process" view with wider distro packaging and no Rust toolchain.
- GyulyVGC/sniffnet — Rust cross-platform network monitor with a GUI and filtering/export;
  use when you want a graphical, actively-developed tool rather than a TUI.
- ntopng — full network monitoring with a web UI, flows, and history; use when you need a
  persistent monitoring system, not a point-in-time view.
- iftop — per-connection bandwidth grouped by host; use when you care about remote hosts and
  don't need process attribution.
- bmon / nload — interface-level throughput graphs; use when you only need aggregate up/down
  per NIC with minimal privileges.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2019-09 | Initial release; repo created 2019-09-06[^2]. Gained early traction via HN and the Rust CLI ecosystem. |
| 0.20.x | 2022 | Later releases added the connection/process/address table split and the `--unit-family` selector seen in current usage[^1]. |
| passive maintenance | 2026 | Feature-frozen; critical fixes only, co-maintainers solicited[^3]. Latest commit 2026-07-06[^2]. |

## References

[^1]: bandwhich README — "How does it work?" and Usage sections. https://github.com/imsnif/bandwhich#how-does-it-work
[^2]: GitHub API, `repos/imsnif/bandwhich` — 11,854 stars, 342 forks, MIT, created 2019-09-06, last push 2026-07-06 (fetched 2026-07-15). https://github.com/imsnif/bandwhich
[^3]: bandwhich README "Project status" and issue #275 "The Future of Bandwhich". https://github.com/imsnif/bandwhich/issues/275

## Tags

rust, cli, tui, networking, bandwidth, monitoring, network-utilization, terminal, packet-capture, per-process, linux
