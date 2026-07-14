# systemd/systemd

> The init system (PID 1) and service manager that most Linux distributions boot into — and the most-argued-over project in the Linux userspace.

[GitHub repo](https://github.com/systemd/systemd) ·
[Official website](https://systemd.io) ·
[License: GPL-2.0](https://github.com/systemd/systemd/blob/main/LICENSE.GPL2)

## Overview

systemd is the process that runs as PID 1 on most mainstream Linux distributions: it brings up the system, supervises services, manages devices, and reaps orphaned processes[^1]. It was started in 2010 by Lennart Poettering and Kay Sievers at Red Hat as a replacement for SysV init and Upstart, and adoption cascaded through the distributions between 2011 and 2015 — Fedora 15 (2011), then RHEL 7, Debian 8, Ubuntu 15.04, Arch, and openSUSE[^2]. By 2015 it was effectively the default init on the Linux desktop and server.

The defining property is that systemd is not one program but a **suite** built around PID 1. The same source tree ships `systemd` (the manager), `journald` (logging), `udev` (device management, merged in 2012), `logind` (seat/session tracking), plus optional daemons for networking (`networkd`), DNS (`resolved`), time sync (`timesyncd`), containers (`nspawn`, `machined`), boot (`systemd-boot`), user home directories (`homed`), and more. Distributions pick which pieces to enable, but the manager, journal, and udev are hard to avoid. This breadth is exactly what makes systemd both convenient and contentious.

The central tension is **scope versus the Unix philosophy**. Supporters point to unified, declarative service definitions, parallelized boot, socket and D-Bus activation, cgroup-based resource control, and consistent tooling across distros. Critics argue it absorbs responsibilities that were previously independent, replaceable tools, produces tight coupling to a single project, and stores logs in a binary format. The dispute was heated enough to produce the Devuan fork of Debian and a wave of drop-in-init alternatives[^3].

## Getting Started

systemd is preinstalled on any systemd-based distribution; you interact with it via `systemctl` and `journalctl`. A minimal unit:

```ini
# /etc/systemd/system/hello.service
[Unit]
Description=Hello web service
After=network.target

[Service]
ExecStart=/usr/local/bin/hello-server --port 8080
Restart=on-failure
DynamicUser=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload          # re-read unit files
sudo systemctl enable --now hello     # start now + start at boot
systemctl status hello                # state, PID, recent log lines
journalctl -u hello -f                # follow this unit's logs
```

## Architecture / How It Works

systemd's model is a dependency graph of **units**, activated to reach **targets** (roughly, runlevels). Unit types include `.service` (processes), `.socket` (activation sockets), `.timer` (cron replacement), `.mount` / `.automount`, `.device`, `.path` (inotify triggers), `.slice` and `.scope` (cgroup grouping), and `.target` (synchronization points). Units are declarative INI-style files; systemd computes ordering from `After=`/`Before=`/`Requires=`/`Wants=` and starts independent branches in parallel.

Core mechanisms:

- **cgroups.** Every service runs in its own control group, so systemd always knows exactly which processes belong to a unit — double-forking daemons cannot escape supervision the way they could under SysV. Resource limits (`MemoryMax=`, `CPUQuota=`, `TasksMax=`) are cgroup properties set per unit.
- **Socket activation.** systemd holds the listening socket and starts the service on first connection, enabling lazy start and boot-time parallelism without ordering the daemons by hand — a model inherited from macOS launchd[^1].
- **D-Bus.** systemd exposes its state and control surface over D-Bus; `systemctl`, `logind`, and desktop environments all talk to it this way.
- **journald.** Logs are written to an indexed **binary** journal with structured metadata (unit, PID, boot ID, priority) rather than plain-text files. `journalctl` queries it; forwarding to a traditional syslog is opt-in.
- **udev.** Device hotplug and `/dev` population run inside the systemd tree since the 2012 merge, which is why udev is hard to use standalone on a systemd host.

The manager, journal, and cgroup coupling are what give systemd its consistency and what critics call its monolithism: the pieces know about each other, so replacing one (e.g. journald with syslog-only) is possible but off the happy path.

## Production Notes

- **Binary logs need journald tooling.** You cannot `grep` the journal directly. `journalctl` is powerful (`-u`, `--since`, `-p err`, `-b -1`, `-o json`) but journald must be running to read logs. Configure `Storage=persistent` and `SystemMaxUse=` in `journald.conf` — the default can either lose logs across reboots (volatile) or grow unbounded, depending on distro.
- **`After=` is ordering, not a readiness guarantee.** `After=network.target` only orders against network *setup*, not actual connectivity; use `network-online.target` (and pull it in with `Wants=`) when a service truly needs a routable address. This is one of the most common misconfigurations.
- **`Type=` mismatches cause hangs.** `Type=simple` assumes the process is up immediately; `Type=notify` requires the daemon to call `sd_notify(READY=1)`; `Type=forking` requires a correct `PIDFile=`. Picking the wrong type produces boot hangs or premature "started" state.
- **Restart loops.** `Restart=always` without `StartLimitIntervalSec=`/`StartLimitBurst=` tuning can thrash a broken service; systemd will eventually rate-limit it into a failed state, which surprises operators expecting infinite retries.
- **cgroup v1 vs v2.** Modern systemd assumes the unified cgroup v2 hierarchy. Older container runtimes and some GPU/monitoring stacks historically needed cgroup v1; mixing them requires kernel boot flags and is a recurring migration headache.
- **Running in containers.** Full systemd inside a container needs cgroup delegation and specific mounts; many base images ship a stub or none at all. `systemd-nspawn` and `machinectl` are the first-party container path, but Docker/Podman/Kubernetes ecosystems dominate and treat PID 1 differently.
- **`DynamicUser=yes` and `systemd-sysusers`** are strong hardening wins (per-service transient UIDs, `ProtectSystem=`, `PrivateTmp=`, seccomp filters) but sandboxing options interact subtly — an over-tight `ProtectHome=`/`ReadOnlyPaths=` set silently breaks services that write outside their expected paths.

## When to Use / When Not

**Use / accept when:**
- You run any mainstream Linux distribution — it is already PID 1 and fighting it costs more than learning it.
- You want declarative service supervision with cgroup limits, socket activation, timers, and sandboxing without assembling separate tools.
- You need consistent service management across Fedora/RHEL/Debian/Ubuntu/Arch/SUSE fleets.

**Look elsewhere when:**
- You are targeting minimal or non-glibc systems (Alpine ships OpenRC/BusyBox init; embedded targets often use BusyBox or runit).
- You want a small, auditable, single-purpose init and reject the suite model on principle.
- You are inside a container and only need one supervised process — a lightweight PID 1 like tini or dumb-init is the right tool, not full systemd.

## Alternatives

- OpenRC — dependency-based init used by Alpine and Gentoo; lighter, script-driven; use when you want a small init without the suite.
- artix-linux / Devuan (runit / s6 / OpenRC spins) — use when you specifically want a systemd-free mainstream distro.
- skarnet/s6 — supervision suite favored for containers and minimal systems; use when you want composable, auditable process supervision.
- BusyBox init — use on embedded/minimal images where footprint dominates.
- Upstart — Canonical's event-based init, now retired; historical predecessor, not a current choice.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1 | 2010-03 | First release at Red Hat (Poettering, Sievers)[^2]. |
| — | 2011 | Fedora 15 ships systemd as default init[^2]. |
| — | 2012 | udev merged into the systemd source tree. |
| — | 2013 | Predictable network interface names introduced. |
| v215 | 2014 | Debian technical committee decision; systemd becomes Debian 8 default[^3]. |
| v220 | 2015 | systemd-boot (from gummiboot) merged. |
| v230 | 2016 | `KillUserProcesses=yes` default flip — sessions killing tmux/screen on logout sparked major backlash[^4]. |
| v245 | 2020 | systemd-homed and systemd-repart introduced[^5]. |
| v247 | 2021 | systemd-oomd (userspace OOM management) added. |

*Versioning is a single monotonically increasing integer; releases land roughly two to four times a year with no semantic-versioning breaks implied by the number.*

## References

[^1]: Lennart Poettering, "systemd" — original announcement, 2010-04-30. http://0pointer.de/blog/projects/systemd.html
[^2]: systemd project documentation and homepage. https://systemd.io/
[^3]: Debian Technical Committee, bug #727708, default init system decision (2014). https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=727708
[^4]: systemd v230 release notes / `KillUserProcesses` change discussion. https://github.com/systemd/systemd/blob/main/NEWS
[^5]: systemd NEWS file (per-release changelog). https://github.com/systemd/systemd/blob/main/NEWS

## Tags

c, linux, init-system, service-manager, pid1, systemd, cgroups, journald, udev, daemon-supervision, systems
