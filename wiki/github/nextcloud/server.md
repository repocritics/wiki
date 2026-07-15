# nextcloud/server

> Self-hosted file sync, share, and collaboration platform — the AGPL fork of ownCloud that became a full "Hub" suite.

[GitHub repo](https://github.com/nextcloud/server) ·
[Official website](https://nextcloud.com) ·
[License: AGPL-3.0](https://github.com/nextcloud/server/blob/master/COPYING)

## Overview

Nextcloud is a PHP application that turns a server you control into a private Google-Drive-plus-Workspace: file storage with sync clients, plus calendars, contacts, mail, video calls, and document editing layered on top through an app ecosystem. It was created in June 2016 when Frank Karlitschek and much of the core team forked ownCloud after leaving the company behind it, and the two codebases still share deep structural DNA[^1]. The pitch is data sovereignty — "a safe home for all your data" — aimed at individuals, homelabbers, and increasingly public-sector and enterprise deployments that cannot or will not put data on US hyperscalers.

The defining tension is scope versus operability. Nextcloud has expanded from a file-sync tool into "Nextcloud Hub"[^2], a bundled collaboration suite (Files, Talk, Groupware, Office via Collabora/OnlyOffice, Assistant/AI). That breadth is the selling point, but the runtime is a large PHP monolith plus dozens of first-party apps, and running it well at more than a handful of users is a real systems-administration job: PHP-FPM tuning, a caching/locking layer (Redis/APCu), background-job scheduling, database tuning, and a correctly configured reverse proxy are all effectively mandatory rather than optional.

Nextcloud is AGPL-3.0-or-later. Nextcloud GmbH sells "Nextcloud Enterprise" (hardened builds, support, longer maintenance windows), but the upstream `server` repo here is the full community product, not a crippled core.

## Getting Started

The path of least resistance for self-hosting is the official All-in-One Docker image, which bundles the database, Redis, and companion containers:

```bash
docker run -it \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  -p 8080:8080 \
  -v nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  nextcloud/all-in-one:latest
```

Then open `https://your-host:8080` to run the setup wizard. For manual installs, drop the release tarball under a PHP-8.x web root, point it at MySQL/MariaDB or PostgreSQL, and drive administration through the `occ` CLI:

```bash
# Run as the web-server user (e.g. www-data)
sudo -u www-data php occ maintenance:install \
  --database pgsql --database-name nextcloud \
  --database-user nc --database-pass secret \
  --admin-user admin --admin-pass secret

sudo -u www-data php occ status
sudo -u www-data php occ app:list
```

`occ` is the real control surface — user management, app installs, background repair steps, and most recovery operations happen there, not in the web UI.

## Architecture / How It Works

At the core is a PHP request-handling framework with a dependency-injection server container, a routing layer, and an app framework. Nearly all functionality — even "core" features like Files and Dashboard — is packaged as an app under `apps/`, loaded through `appinfo/info.xml` manifests. Third-party apps come from the App Store (`apps.nextcloud.com`) and run in the same process with broad access, so app trust is a security boundary, not a sandbox.

Storage is abstracted behind a virtual filesystem. Files can live on local disk or on external backends (S3 as primary "object store" mode, SMB, FTP, other Nextcloud/WebDAV) via the Files External app. File access is exposed over WebDAV; the desktop and mobile sync clients (separate repos) talk WebDAV plus a chunked-upload protocol for large files.

Several subsystems are load-bearing in any non-trivial deployment:

- **Database.** MariaDB/MySQL, PostgreSQL, or SQLite (SQLite only for tiny/test instances). Schema is managed through migration classes; major upgrades run these on first boot.
- **Caching and locking.** Without a memory cache (APCu for local, Redis for distributed) and Redis-backed transactional file locking, concurrent access degrades and the admin panel warns loudly.
- **Background jobs.** Cron, AJAX, or webcron drive periodic work (previews, notifications, federated retries, cleanup). System cron via `cron.php` every 5 minutes is the supported production mode; AJAX mode silently starves under low traffic.
- **Preview generation.** Thumbnails for images, video, and documents can dominate CPU and storage; large photo libraries are a classic resource sink.

Companion features run as their own services: Talk (repo `nextcloud/spreed`) uses a signaling server and an optional High-Performance Backend for scalable calls; document editing shells out to Collabora Online or OnlyOffice document servers. The frontend is Vue-based, built with the `@nextcloud/*` npm libraries, but the server still renders and coordinates most flows.

## Production Notes

**"Install and forget" does not exist.** A default install throws admin warnings until you add APCu/Redis, switch background jobs to system cron, set PHP memory limits, configure the `trusted_domains` and reverse-proxy headers, and often add `default_phone_region` and a proper mail server. Budget time for the Administration → Overview "security & setup warnings" page.

**Upgrades are one-major-version-at-a-time and can be slow.** You cannot skip major versions; going from 27 to 30 means 27→28→29→30, each running migrations. Large databases make the migration step long and, occasionally, fragile — take a full backup (files + database + `config/config.php`) before every upgrade, and expect maintenance-mode downtime. Third-party apps frequently lag a major version and block upgrades until updated.

**Object storage changes the rules.** Running primary storage on S3 improves scaling but means there is no browsable filesystem, previews and encryption behave differently, and some external-storage/versioning assumptions change. Decide primary-vs-external S3 early; migrating storage backends later is painful.

**Scaling is horizontal-ish but stateful.** Multiple PHP app nodes can share a database and Redis, but you need shared storage and a shared Redis for locking/cache coherence. Talk calls at scale require the paid/HPB signaling backend; the built-in signaling is fine only for small groups.

**Performance footguns.** The most common complaints are slow web UI and sync stalls traced back to: missing memory cache, AJAX cron, an undersized database, preview generation on huge folders, or antivirus/mounting on the data directory. `occ` has repair and cleanup commands (`files:scan`, `preview:generate-all`, `db:add-missing-indices`) that are part of routine operation, not emergencies.

**Security posture.** Nextcloud runs a HackerOne bounty program and ships two-factor auth, but the app-in-same-process model means a malicious or vulnerable third-party app is high-impact. Keep the app surface minimal, keep server and PHP patched, and put it behind TLS with sane headers.

## When to Use / When Not

**Use when:**
- You need self-hosted, data-sovereign file sync/share and can staff basic Linux/PHP/DB operations.
- You want an integrated suite (files + groupware + calls + office) under one login and one admin, not a stack of separate tools.
- Compliance, jurisdiction, or cost rules out Google Workspace / Microsoft 365, and on-prem or EU-hosted is a hard requirement.
- You want an extensible platform and are willing to vet third-party apps.

**Avoid when:**
- You want zero-maintenance file sync — a managed provider (or a smaller tool) is less work than operating Nextcloud correctly.
- You need only fast, dumb object storage or a sync engine without the collaboration suite; the monolith is overkill.
- You cannot commit to the one-major-at-a-time upgrade discipline and backups it demands.
- You expect large-scale real-time video without standing up the HPB/signaling infrastructure.

## Alternatives

- owncloud/ocis — the original project's ground-up Go rewrite (oCIS); leaner, microservice-oriented, use when you want ownCloud lineage with a modern backend rather than the PHP monolith.
- seafile/seafile — use when raw file sync speed and reliability matter more than a collaboration suite; block-based storage, C/Python core, fewer groupware features.
- syncthing/syncthing — use when you want peer-to-peer device sync with no central server and no web suite at all.
- filebrowser/filebrowser — use when you just need a lightweight web file manager over existing directories, not sync clients or apps.
- immich-app/immich — use specifically for self-hosted photo/video management, where Nextcloud's preview pipeline struggles at scale.

## History

| Version | Date | Notes |
|---------|------|-------|
| 9 | 2016-06 | Fork of ownCloud announced; version numbered 9 to signal continuity[^1]. |
| 11 | 2017-01 | Security hardening, federated sharing improvements. |
| 13 | 2018-02 | Nextcloud Talk (video/chat) introduced. |
| 15 | 2018-12 | Social (ActivityPub) app, workflow engine. |
| 16 | 2019-04 | ML-based suspicious-login detection, ACLs for groupfolders. |
| 18 | 2020-01 | Rebrand to "Nextcloud Hub" — bundled groupware + office[^2]. |
| 20 | 2020-10 | Unified search, Dashboard start page. |
| 25 | 2022-10 | Hub 3; UI refresh, widgets. |
| 28 | 2023-12 | Hub 7; app store and admin UX updates. |
| 30 | 2024-09 | Ongoing Hub line; continued AI/Assistant integration. |

## References

[^1]: Frank Karlitschek, "Nextcloud — the next chapter" — 2016-06-02. https://nextcloud.com/blog/nextcloud-the-next-chapter/
[^2]: Nextcloud, "Introducing Nextcloud Hub" — 2020-01-17. https://nextcloud.com/blog/introducing-nextcloud-hub/
[^3]: Nextcloud Administration Manual — installation, caching, background jobs, and scaling. https://docs.nextcloud.com/server/latest/admin_manual/
[^4]: Nextcloud App Store. https://apps.nextcloud.com

## Tags

php, self-hosted, file-sync, cloud-storage, collaboration, groupware, webdav, agpl, on-premises, federation, data-sovereignty
