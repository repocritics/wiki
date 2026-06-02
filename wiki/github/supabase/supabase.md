# supabase/supabase

Supabase — the open-source Firebase alternative. Postgres + Auth + Storage + Realtime + Edge Functions + Vector + a polished dashboard.

## What it is

A TypeScript-based backend-as-a-service built around Postgres. Each Supabase project gets: a Postgres database, REST + GraphQL APIs (auto-generated via PostgREST), Auth (email, OAuth, magic links), Storage (S3-compatible), Realtime (Postgres LISTEN/NOTIFY exposed via WebSocket), Edge Functions (Deno-based serverless), Vector storage via pgvector. Cloud-managed at supabase.com; self-hostable via Docker. Apache 2.0 licensed.

## Key features

- Postgres as the canonical data store — no proprietary database.
- Auto-generated REST + GraphQL APIs via PostgREST / pg_graphql.
- Auth: email, magic links, OAuth (many providers), phone OTP, multi-factor.
- Storage: S3-compatible object storage with image transformations.
- Realtime: live subscriptions to DB changes.
- Edge Functions: Deno-based serverless.
- Vector storage via pgvector.
- Self-host or use Supabase Cloud.
- Apache 2.0 licensed.

## Tech stack

- TypeScript primary.
- Postgres at the heart.
- Deno for Edge Functions.
- Distributed via Supabase CLI + Docker Compose for self-host.

## When to reach for it

- You want a Firebase-like BaaS with Postgres ergonomics.
- You want vendor-portable — Supabase's stack is OSS so you can leave the hosted version.
- You need vector search alongside relational data.

## When *not* to reach for it

- You want fully NoSQL — Firebase or Appwrite are closer-fit.
- You don't want to operate a Postgres-flavored backend.

## Maturity signal

Actively maintained under Supabase Inc. 4+ years; rapid feature growth.

## Alternatives

- Firebase — managed-only, Google-owned.
- `appwrite/appwrite` — alternative OSS BaaS.
- PocketBase — single-binary SQLite BaaS.

## Tags

typescript, postgresql, backend-as-a-service, supabase, firebase, self-hosted, apache-license, auth, realtime, vector
