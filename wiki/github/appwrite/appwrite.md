# appwrite/appwrite

A self-hostable backend-as-a-service alternative to Firebase / Supabase — Auth, Databases, Storage, Functions, Messaging, Hosting, Realtime in one Docker-deployable stack.

## What it is

A backend platform that ships the canonical BaaS feature set as an OSS, self-hostable Docker app. Provides per-language SDKs (web, mobile, server) for projects to call its REST + Realtime APIs. Positions itself between Firebase (full-managed, Google-owned) and Supabase (Postgres-centric, OSS) as the BaaS choice for teams that want vendor sovereignty and a managed-feeling DX. Commercial cloud at appwrite.io.

## Key features

- Auth (email/password, OAuth, magic links, anonymous, phone OTP).
- Databases with schema, permissions, indexes (built on MariaDB).
- File storage with encryption-at-rest and image transformations.
- Cloud Functions (serverless code execution) in many languages.
- Realtime subscriptions (WebSocket).
- Messaging (email, SMS, push notifications).
- Hosting for static sites.
- BSD-3-Clause licensed (until recent versions; verify per release).

## Tech stack

- TypeScript primary on the frontend; PHP on the backend API.
- MariaDB for data storage.
- Redis for caching / queues.
- Docker-Compose / Kubernetes deployment.

## When to reach for it

- You want a Firebase-like DX without the Firebase lock-in.
- You're shipping a mobile + web app and need auth + DB + storage + functions out-of-box.
- You want self-host or vendor-portability for a BaaS layer.

## When *not* to reach for it

- You want Postgres specifically — Supabase is closer-fit.
- You want a single primary database with SQL-first ergonomics — Appwrite's data model is document-shaped.
- You want a tiny dependency footprint — Appwrite's full stack is multi-container.

## Maturity signal

56k stars, 5k forks, BSD-3-Clause, actively maintained. 6-year-old project under Appwrite GmbH. Open-issues count of 930.

## Alternatives

- `supabase/supabase` — Postgres-centric direct competitor.
- `pocketbase/pocketbase` — single-binary, SQLite-based BaaS.
- `nhost/nhost` — Hasura + auth + storage stack.
- Firebase — fully-managed, Google-owned.

## Tags

backend-as-a-service, typescript, php, docker, self-hosted, supabase, firebase, framework
