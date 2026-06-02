# hoppscotch/hoppscotch

An open-source API development ecosystem — Postman / Insomnia alternative with web, desktop, and CLI distributions, plus self-hostable for on-prem.

## What it is

A TypeScript / Vue-based API client that supports REST, GraphQL, WebSocket, and other protocols. Designed as a lightweight, browser-first alternative to Postman — no account required, works offline, ships as a PWA. Hosted version at hoppscotch.io; on-prem version self-hostable via Docker; desktop apps via Tauri; CLI for headless testing.

## Key features

- REST, GraphQL, WebSocket, SSE, MQTT, SocketIO protocol support.
- Collections, environments, variables, scripting.
- Multi-platform: web (PWA), desktop (Tauri), CLI.
- On-prem / enterprise distribution for organizations that can't use the cloud.
- Team workspaces (commercial tier).
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Vue.js on the frontend.
- Tauri for desktop builds.

## When to reach for it

- You want a Postman alternative without the desktop app bloat or account requirement.
- You're testing GraphQL APIs — Hoppscotch's GraphQL surface is well-integrated.
- You need on-prem API tooling for compliance.

## When *not* to reach for it

- You're deeply invested in Postman collections — migration is supported but imperfect.
- You want vendor-supported enterprise features at scale — Postman Enterprise has more polish.

## Maturity signal

79k stars, 6k forks, MIT, actively maintained. 5+ years.

## Alternatives

- Postman — commercial, fuller-featured.
- Insomnia (Kong) — comparable OSS-with-paid-tier.
- Bruno — newer git-friendly alternative.
- Thunder Client — VS Code extension.

## Tags

typescript, vue, api, http-client, graphql, websocket, testing, developer-tools, mit-license, self-hosted
