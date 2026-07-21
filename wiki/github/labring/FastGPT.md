# labring/FastGPT

A knowledge-based AI agent platform that pairs out-of-the-box data processing and RAG retrieval with visual workflow orchestration.

## What it is

FastGPT is a platform for building AI agents and question-answering systems on top of LLMs, offering data processing, model invocation, and RAG retrieval without extensive setup. Applications are assembled through visual Flow-based workflow orchestration, so complex conversational and plugin pipelines can be built and debugged in one place. It targets teams that want to stand up knowledge-base-backed assistants and agentic workflows, either self-hosted or via the hosted service.

## Key features

- Application orchestration through visual workflows, including conversational and plugin workflows with basic RPA nodes, agent skill orchestration, user interaction steps, and bidirectional MCP.
- Knowledge base capabilities: reuse and mixing of multiple libraries, chunk editing and deletion, manual/direct-segmentation/QA-split import, and ingestion of txt, md, html, pdf, docx, pptx, csv, and xlsx plus URL reading and CSV batch import.
- Hybrid retrieval with reranking, plus an API-backed knowledge base option.
- Debugging tools: single-point search testing against a knowledge base, citation feedback that can be edited or deleted during chat, full call-chain logs, and application evaluation.
- Operations features: login-free share windows, one-click iframe embedding, unified conversation records with data annotation, and application operation logs.

## Tech stack

- TypeScript on Next.js, organized as a pnpm/Turborepo monorepo (`@fastgpt/app`, `@fastgpt/admin`, `@fastgpt/global`, `@fastgpt/service`).
- Package manager pinned to `pnpm@10.33.4` (`pnpm` 10.x); Node.js `>=20.19.0`.
- Chakra UI (theme typings generated via `@chakra-ui/cli`) and i18next / next-i18next for internationalization.
- MongoDB in the test and tooling path (`mongodb-memory-server`, a `withMongo` test runner); Vitest for testing; ESLint and Prettier for linting/formatting.
- Deployed with Docker Compose (guided install script), or on Sealos; root manifest version 4.0.

## When to reach for it

- Building knowledge-base-driven chatbots and question-answering systems over your own documents.
- Assembling agent and RPA-style workflows visually rather than coding an orchestration layer.
- Self-hosting a RAG platform via Docker or Sealos, or using the hosted cloud version.
- Embedding an assistant into other sites through iframe or login-free share links.

## When *not* to reach for it

- You want to offer the software itself as a multi-tenant SaaS — the license permits direct commercial backend use but prohibits providing SaaS without authorization.
- You need a lightweight library or a single API call; FastGPT is a full platform with MongoDB, a vector store, and multiple services to operate.
- Your workflow is code-first and you would rather express pipelines in a general-purpose framework than a visual Flow editor.

## Maturity signal

FastGPT has been developed since early 2023 and remains actively maintained, with commits pushed the same day as this writing, a very large following, and an unusually high fork count that signals broad self-hosting and derivative use. It is well past early experimentation (root version 4.0) and backed by a substantial contributor community and a commercial offering. The main caveat is licensing rather than upkeep: it ships under a custom FastGPT Open Source License (reported as NOASSERTION), which constrains SaaS resale and should be reviewed before commercial deployment.

## Alternatives

- Dify — use instead when you want a broadly comparable LLMOps platform for apps, RAG, and agents under a more conventional open-source license.
- RAGFlow — use instead when deep document parsing and retrieval quality are the priority over visual workflow orchestration.
- AnythingLLM — use instead when you want a simpler, more self-contained document chatbot to self-host or run on the desktop.
- Langflow — use instead when you prefer a code-adjacent, component-graph builder in the LangChain ecosystem.

## Notes

- The default README is Simplified Chinese, with English, Bahasa Indonesia, Thai, Vietnamese, and Japanese translations available.
- Licensing is the notable constraint: direct commercial use as a backend service is allowed, but offering it as a SaaS requires authorization, and copyright notices must be retained.
- The repository carries extensive internal agent design and skill documentation (`.agents/design`, `.agents/skills`), reflecting an agent-assisted development process, and lists sibling projects FastGPT-plugin, AI Proxy, and Sealos.

## Tags

typescript, nextjs, rag, llm, knowledge-base, workflow, ai-agents, low-code, mcp, self-hosted
