# immich-app/immich

A high-performance, self-hosted photo and video management solution — the leading OSS alternative to Google Photos / iCloud Photos.

## What it is

A self-hostable photo/video library with mobile (iOS + Android via Flutter) and web (SvelteKit) clients. Automatic background backup from phone, face recognition, geocoding, album sharing, AI-powered search (CLIP-based), and selective sync. Built by the Immich team as the Google Photos alternative that ticks the "fully self-hosted, end-to-end open-source, actively maintained" box. AGPL-3.0 licensed.

## Key features

- Automatic mobile-photo backup (iOS, Android) with WiFi-only / background-task options.
- Face recognition, AI-based search (CLIP embeddings).
- Geocoding + map view of photos.
- Albums, shared albums, public links.
- External-library mode for read-only mounts.
- Web (SvelteKit) + mobile (Flutter) + desktop sync clients.
- Multi-user with per-user libraries.
- AGPL-3.0 licensed.

## Tech stack

- TypeScript primary (NestJS API + SvelteKit web).
- Flutter / Dart for mobile.
- PostgreSQL with pgvector for AI search.
- Redis for queues / caching.
- Docker Compose deployment.

## When to reach for it

- You want a fully self-hosted Google Photos alternative.
- You're privacy-sensitive and don't want photos in a cloud you don't control.
- You have a home server / NAS and want a polished photo-management UI.

## When *not* to reach for it

- You want managed cloud photos without operating a server.
- You're allergic to AGPL.
- You want family-sharing across multiple cloud-provider accounts — Immich is single-deployment.

## Maturity signal

102k stars, 6k forks, AGPL-3.0, actively maintained. Open-issues count of 679 reflects the breadth across mobile + web + AI search.

## Alternatives

- PhotoPrism — alternative self-hosted photo library.
- Ente Photos — encrypted alternative with hosted-or-self-host options.
- Google Photos / iCloud / OneDrive — commercial managed.

## Tags

typescript, flutter, photos, self-hosted, nestjs, sveltekit, agpl, mobile, ios, android, ai
