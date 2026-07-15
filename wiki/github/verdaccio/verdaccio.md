# verdaccio/verdaccio

> A lightweight, zero-config Node.js private npm registry and caching proxy that runs from a single process with its own on-disk store.

[GitHub repo](https://github.com/verdaccio/verdaccio) ·
[Official website](https://www.verdaccio.org/) ·
[License: MIT](https://github.com/verdaccio/verdaccio/blob/master/LICENSE)

## Overview

Verdaccio is a private npm registry you can run locally with no external
database and no mandatory configuration. It began in 2017 as a maintained fork
of the abandoned **sinopia** project and kept sinopia's core idea: a single
Node.js process that serves the npm registry API, stores package metadata and
tarballs on the local filesystem, and proxies (uplinks) to upstream registries
like npmjs.org, caching whatever it downloads[^1]. As of 2026 it is the default
answer to "I need a private registry but don't want to stand up Nexus or
Artifactory."

Its two primary jobs are hosting your own private/scoped packages and acting as
a caching proxy in front of the public registry. The caching role is why it is
so widely embedded in CI: projects including create-react-app, pnpm, Babel,
Storybook, Docusaurus, and Angular CLI boot a throwaway Verdaccio instance to
publish and install their own packages during end-to-end tests without touching
the real registry.

The defining tradeoff is **simplicity versus scale**. Verdaccio is deliberately
a single-node, filesystem-backed service — that is what makes it start in
seconds and require zero setup. It is explicitly not built to be a
horizontally-scaled, highly-available public registry replacement. When teams
outgrow the single-process model, the answer is usually a heavier tool, not a
bigger Verdaccio cluster.

## Getting Started

```bash
npm install -g verdaccio   # or: docker run -it --rm -p 4873:4873 verdaccio/verdaccio
verdaccio                  # serves on http://localhost:4873/
```

Point npm at it and publish:

```bash
npm adduser  --registry http://localhost:4873/
npm publish  --registry http://localhost:4873/
npm install  --registry http://localhost:4873/   # falls through to the npmjs uplink on a cache miss
```

A minimal `config.yaml` — the whole behavior of the registry is expressed here:

```yaml
storage: ./storage
auth:
  htpasswd:
    file: ./htpasswd
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
packages:
  '@myscope/*':
    access: $authenticated
    publish: $authenticated
  '**':
    access: $all
    publish: $authenticated
    proxy: npmjs        # cache misses are fetched from the npmjs uplink and stored
```

## Architecture / How It Works

Verdaccio is a Node.js HTTP server (a monorepo of `@verdaccio/*` packages) with
four pluggable extension points: **storage**, **auth**, **middleware**, and
**uplinks**. Everything else is convention.

- **Storage.** The default `local-storage` plugin writes one JSON metadata
  document per package plus the tarball blobs to a directory on disk, and keeps
  a small index file (`.verdaccio-db.json`) listing known packages. There is no
  SQL database. Cloud storage (Amazon S3, Google Cloud Storage) is available as
  separate community plugins that implement the same storage interface[^2].
- **Uplinks.** Each configured upstream registry is an uplink. On a request
  Verdaccio checks local storage first; on a miss it fetches from the uplink,
  streams the tarball to the client, and caches it locally so the second
  install is served from disk. Multiple uplinks can be chained.
- **Packages block.** The `packages` section is an ordered list of glob
  patterns mapping to `access` / `publish` / `unpublish` / `proxy` rules,
  evaluated top-to-bottom. Built-in groups are `$all`, `$anonymous`, and
  `$authenticated`; auth plugins can add named groups/teams.
- **Auth.** The default is an `htpasswd` file. LDAP, GitHub/GitLab OAuth, and
  other backends exist as plugins.
- **Web UI.** A React single-page app served from the same process, backed by
  the registry's own search and metadata endpoints.

The whole thing runs in one process. That is the architecture, not an
implementation detail — the coupling between "single process" and "local
filesystem store" is what defines both its strengths and its limits.

## Production Notes

**It is a single-process service.** Verdaccio does not support PM2 cluster mode;
running it under a clustered process manager can produce undefined behavior,
because multiple workers race on the same on-disk metadata files[^3]. Run one
instance per host.

**Horizontal scaling is not the design.** Putting several Verdaccio instances
behind a load balancer with shared S3/GCS storage does not give you a
consistent cluster — metadata caching and write handling assume a single writer.
For real multi-instance HA you want a purpose-built registry, not more Verdaccio
nodes.

**Disk grows unbounded.** The proxy cache keeps every tarball it has ever
fetched. There is no built-in TTL/eviction for cached upstream packages, so the
storage directory must be monitored and pruned, and backed up if it holds your
private packages (it is your database).

**Auth defaults don't scale to teams.** `htpasswd` is fine for a handful of
users; beyond that, move to an LDAP or OAuth plugin. Token handling and group
mapping quality vary by plugin — vet the specific one you adopt.

**Upgrades have real migration cost.** The 3.x → 4.x jump was a substantial
rewrite: a new React web UI and a changed plugin API, so third-party plugins had
to be ported and some config keys moved[^4]. Later majors mostly raised the
minimum Node.js version (6.x requires Node 18+). Pin the Verdaccio version in CI
images, since a silently newer major can break a previously working config.

**Search and large catalogs.** The built-in search and metadata handling are
tuned for modest private catalogs. Very large proxied namespaces can make the UI
search and some metadata operations sluggish.

**HTTPS and reverse proxies.** In most deployments Verdaccio sits behind nginx /
Traefik / a cloud LB for TLS termination; getting `url_prefix` and the
`X-Forwarded-*` headers right is the usual first-deploy footgun.

## When to Use / When Not

**Use when:**
- You want a private/scoped package registry with near-zero setup.
- You need a caching proxy in front of npmjs for CI speed, offline resilience,
  or protection against upstream outages and left-pad-style incidents.
- You need an ephemeral registry for end-to-end publish/install tests.
- One host is enough and Node.js is already in your stack.

**Avoid when:**
- You need a highly-available, horizontally-scaled registry — Verdaccio is
  single-node by design.
- You want one repository for many ecosystems (Maven, PyPI, Docker, npm) under
  unified RBAC — reach for Nexus or Artifactory.
- You want a fully managed registry with an SLA and don't want to operate
  storage/backups yourself — use a hosted option.

## Alternatives

- sonatype/nexus-public — polyglot artifact repository (npm, Maven, Docker,
  PyPI, more) with enterprise RBAC; use it when one registry must serve many
  ecosystems, and accept the heavier footprint.
- cnpm/cnpmjs.org — Alibaba's private npm registry backed by a real database
  (MySQL/etc.); use it when you need a database-backed, multi-tenant private
  registry rather than a single-file store.
- JFrog Artifactory — commercial polyglot registry; use it when you want a
  managed/enterprise product with support and HA, not a self-run Node process.
- GitHub Packages / npm private packages — hosted registries; use them when you
  already live in that ecosystem and don't want to operate a server at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2017 | Forked from the abandoned sinopia registry; renamed Verdaccio[^1]. |
| 3.x | 2018 | Consolidation of the sinopia-era codebase and config. |
| 4.0 | 2019-08 | Major rewrite: new React web UI and revised plugin API[^4]. |
| 5.0 | 2021 | New storage/HTTP internals; raised minimum Node.js version. |
| 6.x | 2024 | Current stable line; requires Node.js 18 or higher[^5]. |
| next | 2026 | Development branch (`next-9`), targeting newer Node.js runtimes. |

## References

[^1]: Verdaccio — "Introduction" and project origin as a sinopia fork. https://verdaccio.org/docs/what-is-verdaccio
[^2]: Verdaccio docs — storage plugins (local-storage default; S3, Google Cloud Storage). https://verdaccio.org/docs/plugins
[^3]: Verdaccio README / issue #1301 — cluster mode (e.g. PM2 cluster) is not supported. https://github.com/verdaccio/verdaccio/issues/1301
[^4]: Verdaccio 4 — web UI rewrite and plugin API changes. https://verdaccio.org/blog/2019/07/26/verdaccio-4-release
[^5]: Verdaccio README — Version 6 requires Node.js 18 or higher (6.x branch). https://github.com/verdaccio/verdaccio/blob/master/README.md

## Tags

typescript, nodejs, npm, private-registry, registry-proxy, package-manager, devops, ci, docker, self-hosted, yarn, pnpm
