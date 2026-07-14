# portainer/portainer

> A single-container web GUI for managing Docker, Swarm, and Kubernetes environments.

[GitHub repo](https://github.com/portainer/portainer) ·
[Official website](https://www.portainer.io) ·
[License: Zlib](https://github.com/portainer/portainer/blob/develop/LICENSE)

## Overview

Portainer is a container management UI first released in 2016[^1]. It ships as a single container that exposes a web interface over the Docker API (via the local socket or a remote endpoint), letting you inspect and manage containers, images, volumes, networks, stacks, and — since the 2.0 rewrite in 2020 — Kubernetes and Docker Swarm resources[^2]. Its pitch is operational simplicity: one `docker run`, a browser, and you have a dashboard over an orchestrator that would otherwise be driven entirely from the CLI.

The project is split into two editions from one codebase. **Community Edition (CE)**, in this repo, is the free zlib-licensed core. **Business Edition (BE)** is a commercial superset that adds RBAC, external auth (LDAP/OAuth/SAML), registry management, and support[^3]. The defining tension is that the most commonly requested "team" features — granular access control, audit, multiple authenticated users with scoped permissions — live behind the BE license. CE is genuinely useful for a single operator or a trusting team, but the moment you need real multi-tenancy the open-source edition points you at the paid one.

The second defining tension is security. Giving a web app control of the Docker socket is equivalent to giving it root on the host, so where and how you expose Portainer matters more than for most tools of its size.

## Getting Started

```bash
docker volume create portainer_data
docker run -d -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Then open `https://localhost:9443` and set the admin password within the first few minutes — the setup form times out for security and requires a container restart if you miss the window.

For a remote or clustered host, run the **agent** on each node instead of mounting the socket into Portainer directly:

```bash
docker run -d -p 9001:9001 --name portainer_agent --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
```

## Architecture / How It Works

Portainer is a Go backend serving a single-page frontend, packaged as one image. Key pieces:

- **Backend** — a Go HTTP API (the `api/` tree) that proxies to orchestrator APIs. It talks to Docker over the socket/TCP, to Kubernetes via client-go, and to Swarm through the Docker API. It does not run its own scheduler; it is a control-plane UI, not an orchestrator.
- **Datastore** — an embedded BoltDB (bbolt) key-value file at `/data/portainer.db`. This holds users, endpoints, stacks, settings, and RBAC config. There is no external database option; Portainer's own state is single-node and not horizontally replicated.
- **Frontend** — historically AngularJS, being migrated to React over the 2.x line; both coexist in `app/`. The React client consumes a TypeScript API SDK generated from the Go API's Swagger annotations (`make generate-api`), so the frontend types track the backend contract rather than being hand-written[^4].
- **Environments (endpoints)** — each managed host/cluster is an "environment." Portainer reaches them one of three ways: the local Docker socket, a direct TCP/TLS connection, or the **agent**. The agent adds cluster awareness (routing requests to the right node in Swarm) and volume browsing.
- **Edge agents** — for hosts behind NAT/firewalls, the edge agent dials *out* to Portainer and holds a reverse tunnel, so Portainer never needs inbound reachability to the remote host. This is how Portainer manages fleets of edge/IoT devices.

The coupling story: Portainer is a thin authenticated proxy plus a state DB. Almost everything it shows is a live call to the underlying orchestrator, which is why it feels fast and why it inherits the orchestrator's permission model. The stateful parts it owns (users, stack definitions, access policies) are the parts that don't survive losing `/data`.

## Production Notes

- **The socket is root.** Mounting `/var/run/docker.sock` gives Portainer — and anyone who compromises it — full control of the host. Do not expose the UI to the public internet on a socket-mounted deployment. Prefer the agent, terminate TLS, and put it behind a VPN or authenticated reverse proxy.
- **No HA for Portainer itself.** The BoltDB datastore is a single file; you cannot run two active Portainer replicas against shared state. Treat it as a single instance and back up `/data` (or the `portainer_data` volume). Restores are a file copy.
- **CE vs BE feature cliffs.** RBAC, teams-with-real-permissions, external identity providers, and registry management are BE-only. Building a workflow in CE and later discovering the access-control piece needs a license is a common surprise. Read the CE/BE feature table before committing a team to CE.
- **Stacks are not Compose-native long-term state.** Portainer deploys Compose/stack files but stores its own copy; editing the underlying stack outside Portainer (or via GitOps) can drift from what Portainer thinks is deployed. The "Git repository" stack option and webhook redeploy exist to manage this, but the two-sources-of-truth risk is real.
- **Version support window.** Portainer officially supports only "current minus two" Docker versions; older Docker daemons may work but are unsupported[^5]. Kubernetes support similarly tracks recent releases.
- **Analytics on by default.** CE sends anonymous usage telemetry (Matomo) unless you disable it on first launch[^6].
- **Upgrades.** Upgrading is usually pulling a new image and recreating the container against the same volume; Portainer migrates the BoltDB schema on start. Downgrades across a schema migration are not supported — back up `/data` before a major upgrade.

## When to Use / When Not

**Use when:**
- You want a GUI over Docker/Swarm on one or a few hosts without learning the full CLI surface.
- You manage edge/remote hosts behind NAT and want a pull-based tunnel model.
- A small team needs a shared, visual view of what's running.
- You want quick Compose/stack deploys with a redeploy-on-git-push option.

**Avoid when:**
- You need fine-grained RBAC, SSO, or audit on the free edition — that is BE.
- You're running Kubernetes at scale and want a K8s-native platform; Portainer's K8s view is a convenience layer, not a full cluster-management product.
- You want GitOps as the single source of truth — a UI that also owns deploy state fights that model.
- The host is internet-exposed and you can't isolate the socket-privileged UI.

## Alternatives

- rancher/rancher — full Kubernetes management platform; use it when you're all-in on K8s at scale and need cluster provisioning, not just a dashboard.
- louislam/dockge — lightweight Compose-stack-focused UI; use it when you only manage `docker compose` files on one host and want them to stay plain files.
- lensapp/lens — desktop Kubernetes IDE; use it when you want a local client for clusters rather than a hosted web app over Docker.
- kubernetes/dashboard — the official Kubernetes web UI; use it when your world is only K8s and Docker/Swarm are irrelevant.
- coollabsio/coolify — self-hosted PaaS; use it when you want app deployment and build pipelines, not raw container administration.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-06 | First release. AngularJS UI over the Docker API[^1]. |
| 1.11–1.24 | 2017–2020 | Swarm support, stacks, agent, edge agent, endpoint groups. |
| 2.0 | 2020-08 | Major rewrite: Kubernetes support, edge compute, CE/BE split formalized[^2]. |
| 2.x | 2021–2026 | Ongoing React migration, generated API SDK, edge/GitOps features, regular releases every couple of months. |

## References

[^1]: Portainer — project history and origin, portainer.io. https://www.portainer.io
[^2]: Portainer 2.0 release — Kubernetes support and edition split. https://www.portainer.io/blog/portainer-2-0-is-here
[^3]: Portainer CE vs BE feature comparison. https://www.portainer.io/features
[^4]: Portainer README — "Generating API types" (`make generate-api`, Swagger → OpenAPI → hey-api SDK). https://github.com/portainer/portainer/blob/develop/README.md
[^5]: Portainer README — "Limitations": supports current minus two Docker versions. https://github.com/portainer/portainer/blob/develop/README.md
[^6]: Portainer README — "Privacy": Matomo analytics, opt-out on first start. https://github.com/portainer/portainer/blob/develop/README.md

## Tags

docker, kubernetes, docker-swarm, container-management, devops, self-hosted, web-ui, go, typescript, infrastructure, orchestration
