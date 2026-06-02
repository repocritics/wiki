# fastapi/full-stack-fastapi-template

A production-ready full-stack project template — FastAPI backend, React frontend, SQLModel ORM, Postgres, Docker, GitHub Actions, automatic HTTPS.

## What it is

A scaffold for full-stack web apps that combines FastAPI (Python async backend), React + TanStack (TypeScript frontend), SQLModel (Pydantic-flavored ORM over SQLAlchemy), Postgres, Docker Compose, and JWT auth into a runnable project. Maintained by Sebastián Ramírez (FastAPI's author) as the canonical "start here" template for the FastAPI stack.

## Key features

- FastAPI backend with SQLModel ORM, automatic OpenAPI docs, async by default.
- React + TanStack Query + TanStack Router on the frontend.
- Chakra UI components.
- JWT auth with refresh tokens.
- Docker Compose for local development + production.
- Traefik reverse proxy with auto-HTTPS via Let's Encrypt.
- GitHub Actions CI for tests + builds.
- MIT-licensed.

## Tech stack

- Python (FastAPI, SQLModel, Pydantic) on the backend.
- TypeScript (React, TanStack) on the frontend.
- PostgreSQL.
- Docker / Docker Compose.
- Traefik for reverse-proxy + auto-TLS.

## When to reach for it

- You want a production-baseline scaffold for a Python + React full-stack project.
- You're building an internal tool and want auth + DB + frontend wired up.

## When *not* to reach for it

- You want a thinner template — many lighter "fastapi-starter" repos exist.
- You're committed to a different ORM (SQLAlchemy direct, Tortoise, Prisma).

## Maturity signal

43k stars, 9k forks, MIT, actively maintained. Open-issues count of 24 is exceptionally low — tight maintainer triage. This template is the closest thing to a "rails new"-equivalent for FastAPI users.

## Alternatives

- T3-stack / Next.js / Remix for JS-monorepo full-stack.
- Django + DRF + React for Django-flavored full-stack.

## Tags

python, fastapi, react, typescript, postgresql, docker, full-stack, template, mit-license
