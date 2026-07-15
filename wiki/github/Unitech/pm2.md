# Unitech/pm2

> A daemonized process manager for Node.js that keeps apps alive, clusters them across CPU cores, and load-balances between the workers.

[GitHub repo](https://github.com/Unitech/pm2) ·
[Official website](https://pm2.keymetrics.io/) ·
License: AGPL-3.0[^1]

## Overview

PM2 is a long-running process manager for Node.js (and, more recently, Bun)
applications, first published to npm in 2013[^2]. Its job is the unglamorous
middle layer between "I have a `server.js`" and "it runs in production": it
daemonizes the app, restarts it when it crashes, captures stdout/stderr to log
files, and — its headline feature — forks the app across multiple CPU cores
using Node's `cluster` module while load-balancing incoming connections between
the workers. For roughly a decade it was the default answer to "how do I run
Node in production without writing a systemd unit."

The defining tension is that PM2 predates and overlaps with two things that have
since become standard: init systems (systemd) and container orchestrators
(Docker, Kubernetes). PM2 supervises processes; so does systemd; so does the
container runtime restart policy. Running PM2's clustering *inside* a container
is a common anti-pattern — you end up with a process manager managing processes
inside a scheduler that already manages processes. PM2 remains genuinely useful
on plain VMs and bare metal, which is exactly the deployment shape that has been
shrinking. It is widely deployed and still maintained, but the ground under its
core use case has moved.

PM2 is also the on-ramp to Keymetrics/PM2+, a commercial monitoring SaaS from
the same authors[^3]. The open-source CLI is complete on its own, but much of
the documentation nudges toward the paid dashboard.

## Getting Started

```bash
npm install pm2 -g
```

```bash
# Start an app and daemonize it
pm2 start app.js --name api

# Cluster mode: one worker per CPU core, load-balanced
pm2 start app.js -i max

# Inspect, tail logs, live dashboard
pm2 list
pm2 logs api
pm2 monit

# Zero-downtime reload (rolling restart of cluster workers)
pm2 reload api

# Persist the current process list and regenerate on boot
pm2 startup      # prints an init-system command to run once
pm2 save
```

An `ecosystem.config.js` file is the declarative form and the recommended way to
pin configuration in a repo:

```javascript
module.exports = {
  apps: [{
    name: "api",
    script: "./server.js",
    instances: "max",
    exec_mode: "cluster",
    max_memory_restart: "512M",
    env: { NODE_ENV: "production" },
  }],
};
```

## Architecture / How It Works

PM2 runs as two parts. The **CLI** you invoke (`pm2 start …`) is a short-lived
client. On first use it spawns a persistent background **God daemon**
(`God.js`), and the two communicate over a local socket using RPC. The daemon is
what actually owns your processes; the CLI just sends it commands. This is why
`pm2 list` from any shell sees the same processes, and why killing your terminal
does not kill the apps.

Cluster mode is a wrapper over Node's built-in `cluster` module. PM2 forks N
copies of your app; the Node cluster master distributes incoming socket
connections across the workers (round-robin on most platforms). Crucially, this
only load-balances at the **connection** level and only works when your app
listens on a socket — it is not an HTTP-aware reverse proxy and does no
path-based routing, TLS termination, or health-check-driven draining. "Load
balancer" in PM2's description means kernel-assisted connection distribution
between identical local workers, nothing more.

**Zero-downtime reload** works by restarting cluster workers one at a time,
waiting for each new worker to signal ready before killing the old one. This
depends on the app cleanly closing its listening socket and finishing in-flight
requests on `SIGINT`; apps that ignore graceful-shutdown signals will drop
connections during a "zero-downtime" reload despite the name.

Process state is serialized to `~/.pm2/` — `dump.pm2` holds the saved process
list, plus per-app log files and PID files. The `pm2 startup` command generates
an init-system script (systemd, upstart, launchd, openrc, and others) that
re-launches the daemon on boot; `pm2 save` writes the dump the daemon restores
from. If you forget `pm2 save`, a reboot brings back an empty daemon.

## Production Notes

- **Do not run cluster mode inside a container.** Give each container one
  process and let the orchestrator scale replicas. PM2 clustering plus
  Kubernetes replicas multiplies your worker count in ways that are hard to
  reason about and defeats per-pod resource limits. If you use PM2 in Docker at
  all, use `pm2-runtime` (a foreground, non-daemonizing entrypoint) in fork mode
  so the container has a proper PID 1 that exits when the app dies.
- **The daemon is a single point of failure and state.** A corrupted
  `~/.pm2/dump.pm2`, a PM2 version mismatch after an upgrade, or a wedged daemon
  can require `pm2 kill` (which stops *all* managed apps). `pm2 update` exists
  specifically because upgrading the CLI without reloading the daemon leaves the
  two on different versions.
- **Log files grow unbounded by default.** PM2 does not rotate logs out of the
  box; you must `pm2 install pm2-logrotate` (a separate module) or ship logs
  elsewhere. Unbounded `~/.pm2/logs` filling the disk is a classic PM2 outage.
- **`max_memory_restart` is a blunt instrument.** It kills and restarts a worker
  when RSS crosses a threshold — useful as a leak backstop, but it drops in-flight
  requests on that worker and masks rather than fixes the leak.
- **Cluster mode requires a stateless app.** Any in-memory session, cache, or
  WebSocket affinity breaks once connections spread across workers. You need a
  shared store (Redis) and sticky sessions handled upstream.
- **Metrics beyond CPU/memory push you toward PM2+.** The free `pm2 monit` and
  `pm2 describe` cover basics; deeper APM, transaction tracing, and cross-server
  dashboards are the paid product.

## When to Use / When Not

**Use when:**
- You deploy Node.js to plain VMs or bare metal and want restart-on-crash,
  clustering, and log capture without writing systemd units.
- You want zero-downtime reloads for a stateless HTTP service on a single host.
- You need a quick, scriptable way to run and inspect several Node processes on
  one machine.

**Avoid when:**
- You run in containers/Kubernetes — the runtime's restart policy and the
  orchestrator's scaling already do PM2's job; use `pm2-runtime` fork mode at
  most, or nothing.
- Your app is stateful in-memory and cannot be safely forked across workers.
- You want HTTP-aware load balancing, TLS, or health checks — reach for a real
  reverse proxy (nginx, HAProxy, Traefik).

## Alternatives

- nodejs/node `cluster` — PM2's clustering is a wrapper over this; if you only
  need multi-core forking and already have supervision, use it directly.
- systemd — use when you're on Linux and want the OS to supervise, restart, and
  boot your service with no extra runtime.
- Docker/Kubernetes — use when you want container-level restart policies and
  replica scaling instead of an in-host process manager.
- foreverjs/forever — use when you want a minimal keep-alive supervisor without
  clustering or a dashboard.
- godaddy/node-cluster or nodejs/pm2-alternatives aside, strong-pm (deprecated)
  and nodemon (dev-only, not production) round out the historical field.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2013-06 | Initial public releases; keep-alive + log capture[^2]. |
| 2.0 | 2016-08 | Rewrite; ecosystem file, namespaced process management. |
| 3.0 | 2018-05 | Redesigned internals, improved cluster/reload handling. |
| 4.0 | 2019-08 | `pm2-runtime` container focus, updated monitoring hooks. |
| 5.0 | 2021-09 | Dependency modernization, security fixes, Node 16+ baseline. |
| 6.0 | 2025 | Bun support, newer Node baselines[^4]. |

## References

[^1]: GitHub reports the license as `NOASSERTION` (custom/dual terms); the
    project README states PM2 is made available under the GNU Affero General
    Public License 3.0, with other licenses available on request from
    Keymetrics. https://github.com/Unitech/pm2/blob/master/README.md
[^2]: `pm2` on the npm registry — first published 2013. https://www.npmjs.com/package/pm2
[^3]: PM2+ / Keymetrics monitoring dashboard. https://app.pm2.io/
[^4]: PM2 CHANGELOG. https://github.com/Unitech/pm2/blob/master/CHANGELOG.md

## Tags

javascript, nodejs, process-manager, cluster, load-balancer, devops, production, monitoring, cli, daemon, bun
