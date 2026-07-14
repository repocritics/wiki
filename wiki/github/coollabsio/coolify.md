# coollabsio/coolify

> A self-hostable PaaS that turns your own servers into a Heroku/Vercel-style deploy target over nothing but SSH.

[GitHub repo](https://github.com/coollabsio/coolify) ·
[Official website](https://coolify.io) ·
[License: Apache-2.0](https://github.com/coollabsio/coolify/blob/v4.x/LICENSE)

## Overview

Coolify is an open-source control plane for self-hosting applications, databases, and third-party services on hardware you own. You point it at a server over SSH — a VPS, bare metal, or a Raspberry Pi — and it installs Docker, wires up a reverse proxy with automatic TLS, and gives you a git-push / one-click deploy workflow that resembles Heroku, Netlify, or Vercel[^1]. It was created by Andras Bacsai (coollabs) in 2021 and is maintained primarily by him and peaklabs-dev[^2].

The project's core pitch is the absence of vendor lock-in: every application, database, and proxy config lives on your server as plain Docker resources, so if you stop using Coolify the containers keep running — you only lose the automation layer, not the workloads[^1]. This is the honest tension of the product. Coolify is a single, opinionated orchestration server that installs and manages a lot of moving parts on your behalf; that convenience is real, but it also means the Coolify host becomes a stateful piece of infrastructure you now have to operate, back up, and upgrade. Coolify sits between "raw Docker + a reverse proxy you hand-roll" and "a managed cloud PaaS you pay per-seat for."

The codebase is a Laravel (PHP) application. There is also a paid Cloud offering (app.coolify.io) that runs the same software with high-availability and managed notifications, which funds development[^1]. As of this writing the repo has ~58.5k stars and ~5k forks, with an active issue tracker (~800 open) and near-daily commits on the `v4.x` branch — this is a fast-moving project, not a frozen one.

## Getting Started

The supported install path is a root shell script that provisions Docker and starts Coolify's own stack (app, PostgreSQL, Redis, realtime) as containers on the host[^3]:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

After install, Coolify serves a web UI on port 8000. You then:

1. Add a server (localhost or a remote host reachable by SSH key).
2. Create a project/environment, then add a resource — a git repository, a Docker image, a `docker-compose.yaml`, or one of the 280+ prebuilt service templates.
3. Set a domain; Coolify requests a Let's Encrypt certificate through its proxy automatically.

Git-based apps deploy on push via webhook (GitHub App, deploy key, or GitLab/Gitea/Bitbucket integration). Builds default to Nixpacks buildpacks[^4], with Dockerfile and Docker Compose as alternatives.

## Architecture / How It Works

Coolify is a monolithic Laravel app that acts as an orchestrator; it does not run your workloads itself. The real work happens over SSH: queued background jobs connect to each managed server and execute `docker` / `docker compose` commands to build images and start containers.

Key pieces:

- **Control-plane stack.** The Coolify server runs itself in Docker: the PHP/Laravel app, a PostgreSQL database for its own state, Redis for queues and real-time state, and a realtime/websocket service for live logs and terminal streaming. Queue workers (Laravel's queue system) drive deployments asynchronously.
- **Managed servers.** Each target server needs only Docker and SSH access. Coolify stores connection details and pushes work to it; there is no long-running agent installed on managed hosts — it is SSH-driven.
- **Reverse proxy.** Traefik is the default edge proxy (Caddy is also supported), providing routing and automatic HTTPS via Let's Encrypt. Coolify generates the proxy config from your domain and port settings[^5].
- **Builders.** Nixpacks (originally from Railway) auto-detects language and produces an image without a Dockerfile; you can override with your own Dockerfile, a Compose file for multi-container apps, or a static-site buildpack.
- **UI.** The interface is server-rendered and real-time; historically Livewire-driven, with parts of the newer UI moving toward an Inertia + Svelte stack (reflected in the repo topics).

Because everything a resource needs is expressed as Docker config saved on the server, Coolify's own database is metadata plus automation, not the source of truth for whether your app runs.

## Production Notes

- **The Coolify host is a single point of control, and self-hosted has no built-in HA.** If the Coolify server goes down, already-running app containers keep serving, but you lose deployments, webhooks, scheduled backups, and the dashboard until it recovers. High-availability is a Cloud-only feature[^1]. Treat the control server as production infrastructure.
- **Run Coolify on a dedicated server.** The recommended topology is one server for Coolify and one or more separate servers for your resources[^1]. Colocating heavy workloads with the control plane leads to resource contention and makes upgrades riskier.
- **Back up Coolify itself.** Coolify can schedule Postgres/MySQL backups for your databases (including to S3-compatible storage), but backing up Coolify's own state (its Postgres DB and the `/data/coolify` directory holding SSH keys, proxy config, and env vars) is your responsibility. Losing it means re-adding servers and secrets by hand.
- **Fast release cadence, occasional breakage.** Coolify ships frequently. Read release notes before upgrading; pin/snapshot the host first. Historically some minor releases have required manual intervention or caused proxy/regeneration hiccups.
- **The install script is opinionated and root-level.** It installs Docker, configures the host, and expects a fairly clean machine. Running it on a server with a pre-existing custom Docker/reverse-proxy setup can conflict.
- **Secrets live on the server in plaintext-adjacent form.** Environment variables and SSH keys are stored so Coolify can use them non-interactively; this is fine for self-hosting but means host compromise exposes them. Restrict access to the Coolify server accordingly.
- **Teams exist, but this is not hardened multi-tenancy.** Coolify has teams and roles for collaboration, not the audit/RBAC isolation you would expect from an enterprise platform. Do not treat it as a way to safely sandbox untrusted users.

## When to Use / When Not

**Use when:**
- You want Heroku-like developer experience on your own VPS/bare metal/homelab and want to control cost and data.
- You are comfortable operating one stateful server and want to avoid per-seat cloud PaaS pricing.
- You need one-click databases and common services (Postgres, Redis, MySQL, and many app templates) without composing them by hand.
- Avoiding vendor lock-in matters: you want configs to remain plain Docker resources.

**Avoid when:**
- You need turnkey high availability and zero-ops without running the control plane yourself (use a managed cloud, or Coolify Cloud).
- You need Kubernetes-scale scheduling, autoscaling, or bin-packing across a large fleet.
- You require mature RBAC, audit logs, and tenant isolation for compliance.
- You want a slow-moving, rarely-updated tool; Coolify iterates quickly.

## Alternatives

- dokploy/dokploy — newer self-hostable PaaS built on Docker Swarm; use when you want Coolify-style UX with native multi-node clustering out of the box.
- caprover/caprover — older Docker Swarm PaaS with a large template library; use when you want a longer-established, Swarm-native option.
- dokku/dokku — the minimal, CLI-first Heroku-in-a-box for a single server; use when you prefer a lean buildpack tool without a web dashboard.
- portainer/portainer — a Docker/Kubernetes management UI, not a git-push PaaS; use when you want to manage containers directly rather than deploy from source.
- railwayapp/railway — the managed cloud whose Nixpacks builder Coolify uses; use when you would rather pay for a hosted platform than self-host at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2021-01 | Project started by Andras Bacsai; first public release later in 2021[^2]. |
| v2–v3 | 2021–2022 | Earlier generations built on a Node.js + Svelte + Prisma stack. |
| v4 | 2023–2024 | Complete rewrite in PHP/Laravel; current line, `v4.x` default branch[^6]. |
| Cloud | ongoing | Paid managed offering (app.coolify.io) funds development[^1]. |

## References

[^1]: Coolify README and landing page — "An open-source & self-hostable Heroku / Netlify / Vercel alternative." https://coolify.io
[^2]: Core maintainers: Andras Bacsai (coollabs) and peaklabs-dev. https://github.com/andrasbacsai
[^3]: Installation instructions and script source. https://coolify.io/docs/installation
[^4]: Nixpacks buildpack, originally from Railway, used as Coolify's default builder. https://nixpacks.com
[^5]: Coolify uses Traefik (default) / Caddy as the edge proxy with Let's Encrypt TLS. https://coolify.io/docs
[^6]: `v4.x` is the repository default branch; v4 is a rewrite from the earlier Node.js-based generations. https://github.com/coollabsio/coolify

## Tags

php, laravel, self-hosted, paas, docker, deployment, devops, self-hosting, heroku-alternative, reverse-proxy, open-source
