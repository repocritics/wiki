# n8n-io/n8n

A fair-code workflow automation platform — Zapier-style visual building with the option to drop into custom code, plus 400+ integrations and native AI.

## What it is

A workflow engine that lets technical teams compose automations as visual node graphs and, when the visual layer hits its limits, switch any node to TypeScript/Python code. Targets the iPaaS niche (integration Platform-as-a-Service) but distributes the runtime so customers can self-host instead of being locked to a SaaS account. The "fair-code" license is a custom non-OSI license: the source is open, but commercial-use restrictions apply. AI capability is first-party: MCP client + MCP server roles, plus LLM-aware nodes.

## Key features

- 400+ integrations spanning SaaS APIs, databases, message buses, and AI services.
- Visual workflow builder with the per-node "switch to code" escape hatch (TypeScript or Python).
- Self-host or n8n.io cloud — same runtime in either deployment.
- Native MCP (Model Context Protocol) client + server — n8n workflows can both consume MCP tools and expose themselves as MCP tools.
- AI-aware nodes for LLM orchestration, embeddings, vector stores, and agent loops.
- CLI for headless workflow execution and CI.

## Tech stack

- TypeScript primary across the editor (Vue-based), the workflow runtime (Node.js), and the CLI.
- Workflow engine implemented as a Node.js process; persistence to Postgres/SQLite.
- Docker-first deployment for self-hosting; Helm charts for Kubernetes.

## When to reach for it

- You want Zapier-class workflow automation without per-step usage pricing or vendor lock-in.
- Your workflows mix SaaS API calls with custom transformation logic that doesn't fit no-code constraints.
- You need on-premise / VPC-only automation for data-residency reasons.

## When *not* to reach for it

- You're sensitive to non-OSI licenses — "fair-code" is open-source-ish, not OSI-approved; review the LICENSE before commercial deployment.
- You want a fully managed solution without operating the runtime — n8n.io cloud exists but is the secondary path.
- You want a purely code-first automation experience — Temporal, Inngest, or plain queue workers may fit better.

## Maturity signal

190k stars, 58k forks, last push the morning this page was generated. 6-year-old project with strong commercial backing (n8n GmbH) and a growing enterprise customer base. The 1,500 open-issues count reflects the breadth of integration surface; per-integration bug triage is the dominant maintenance burden. The fair-code license has been the recurring debate topic in the OSS community — it works for n8n's commercial model but isn't a drop-in OSI alternative.

## Alternatives

- Zapier — use when you want a fully-managed SaaS with no operating burden and don't need code escape hatches.
- Temporal — use when you want durable execution / saga primitives rather than visual workflow building.
- Apache Airflow — use when you want data-pipeline-flavored DAGs with strong Python ecosystem.
- Inngest — use when you want code-first event-driven workflows with no visual editor.

## Notes

The "fair-code" license is the most important non-obvious fact about this project; read it before betting on it for commercial automation. The MCP integration is unusually first-class — n8n positions itself as an MCP-aware automation hub rather than a generic workflow runner. The 400+ integration count is real but uneven in depth: top-tier integrations (Slack, Google Workspace, AWS) are full-featured; long-tail integrations may be community-contributed.

## Tags

workflow-automation, low-code, self-hosted, typescript, integration, model-context-protocol, ipaas, artificial-intelligence, fair-code
