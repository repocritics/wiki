# twentyhq/twenty

Twenty — an open-source modern CRM. Salesforce / HubSpot alternative with developer-first ergonomics and a polished, Linear-feel UI.

## What it is

A TypeScript CRM that manages the standard sales surface — companies, people, deals, activities, tasks — with a UX that feels closer to Linear or Notion than to Salesforce. Self-hostable via Docker; commercial cloud at twenty.com. Built around a clean GraphQL + REST API surface so custom integrations don't need to fight a legacy schema. AGPL-3.0 licensed.

## Key features

- Standard CRM objects: companies, people, deals, activities, tasks, notes.
- Custom objects + custom fields without leaving the admin UI.
- GraphQL + REST APIs auto-generated against your schema.
- Kanban, table, gallery, and detail views per object.
- Workflows (automation triggers) for status changes, follow-ups, integrations.
- Workspaces with role-based permissions.
- Self-host via Docker or use Twenty Cloud (managed offering).
- Polished, Linear-style UI — significantly less "enterprise-y" than Salesforce / HubSpot.
- AGPL-3.0 licensed.

## Tech stack

- TypeScript primary across frontend + backend.
- NestJS on the backend with Apollo for GraphQL.
- React + Recoil/Jotai on the frontend.
- Postgres for persistence + Redis for queues / caching.
- Docker Compose for local dev + production deploy.

## When to reach for it

- You want a CRM with modern DX and don't need Salesforce's massive integration ecosystem.
- You need self-hosting for data-residency / compliance reasons.
- You want the GraphQL API to ship custom integrations without wrestling a legacy SOAP / REST surface.
- Your team likes Linear's UX and wants the same feel in CRM tooling.

## When *not* to reach for it

- You're allergic to AGPL — the copyleft pulls SaaS hosting into source-release obligations.
- You need the broad Salesforce / HubSpot integration ecosystem (AppExchange, Zapier-anchor for CRM, deep Microsoft Office integration).
- You want vendor-supported with enterprise SLAs at scale.

## Maturity signal

Actively maintained under Twenty PBC. The Y Combinator-backed company funds ongoing development. CRM is a mature product category; differentiation is mostly UX + DX, both of which Twenty invests in. Self-hosting + commercial cloud signals the standard commercial-OSS playbook.

## Alternatives

- Salesforce, HubSpot, Pipedrive — commercial managed.
- EspoCRM, SuiteCRM — older OSS CRMs with deeper ecosystems but dated UX.
- Attio — modern commercial alternative with similar UX philosophy but closed-source.
- Notion / Airtable with custom views — for very small teams.

## Notes

The "modern UX in a CRM" framing is the product's most-distinctive claim — most OSS CRMs (EspoCRM, SuiteCRM) inherit the 2010s-era enterprise look. Twenty pairs that UX with a code-quality backend (NestJS + Postgres + GraphQL) rather than the PHP/MySQL stack typical of older OSS CRMs. AGPL means commercial hosting requires source release; that's intentional and protects the Twenty Cloud commercial offering.

## Tags

typescript, crm, self-hosted, agpl, react, nestjs, postgresql, graphql, framework
