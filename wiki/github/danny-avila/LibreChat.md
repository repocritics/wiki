# danny-avila/LibreChat

A self-hosted, open-source AI chat platform that unifies many model providers behind one ChatGPT-style interface with agents, tools, and multi-user auth.

## What it is

LibreChat is a self-hosted web application for chatting with AI models from a single, privacy-focused interface. It connects to major providers and any OpenAI-compatible endpoint, and adds agents, Model Context Protocol (MCP) support, artifacts, a code interpreter, web search, and enterprise-style multi-user authentication on top of plain chat. It is for teams and individuals who want to run their own AI chat infrastructure rather than depend on a single hosted product.

## Key features

- Model selection across Anthropic, AWS Bedrock, OpenAI, Azure OpenAI, Google, Vertex AI, and the OpenAI Responses API, plus custom endpoints for any OpenAI-compatible API and local/remote providers such as Ollama, Mistral, Groq, and OpenRouter.
- A sandboxed Code Interpreter API that runs Python, Node.js, Go, C/C++, Java, PHP, Rust, and Fortran with file upload and download.
- Agents and tools: no-code custom assistants, an agent marketplace, reusable `SKILL.md` bundles, subagents with isolated context windows, and MCP server support.
- Generative UI with code artifacts (React, HTML, Mermaid), plus image generation and editing via GPT-Image-1, DALL·E, Stable Diffusion, Flux, or MCP servers.
- Resumable streams that reconnect dropped responses and sync across tabs and devices, production-ready with Redis for horizontally scaled deployments.
- Multi-user secure authentication (OAuth2, LDAP, email), a browser-based admin panel for users/roles/config, message search, import/export, speech-to-text and text-to-speech, and a multilingual UI.

## Tech stack

- Primary language: TypeScript (with JavaScript across the API), organized as an npm workspaces monorepo covering `api`, `client`, and `packages/*`.
- Package manager pinned to `npm@11.13.0`, builds orchestrated with Turbo, end-to-end tests with Playwright, and linting/formatting via ESLint and Prettier.
- Search indexing via Meilisearch (present in the API config), and Redis for resumable streams and scaled deployments.
- Ships with Dockerfiles and Docker Compose stacks, with one-click deploy templates for Railway, Zeabur, and Sealos.
- License: MIT. Repository version reported as `v0.8.7`.

## When to reach for it

- Running a shared, self-hosted chat platform for a team that needs multi-user auth, roles, and an admin panel.
- Consolidating many model providers, including local and OpenAI-compatible endpoints, behind one UI.
- Building and sharing agents that use tools, MCP servers, file search, and code execution.
- Keeping conversation data and AI infrastructure under your own control for privacy or compliance reasons.

## When *not* to reach for it

- You just want a personal desktop chat app with minimal setup; a full self-hosted stack with database, search, and optional Redis is more than that requires.
- You are content with a single hosted assistant and do not need provider switching, self-hosting, or multi-user features.
- You lack the operational capacity to deploy and maintain a Dockerized, multi-service application.

## Maturity signal

LibreChat has been developed since early 2023, carries a very large star count, and shows a push date within the last day, all of which point to an actively developed, widely adopted project. A sizable open-issue backlog is expected for software of this breadth and user base rather than a sign of neglect. The MIT license is permissive, the repository is not archived, and the README stresses consulting the changelog for breaking changes before updating — consistent with a fast-moving, feature-rich codebase.

## Alternatives

- Open WebUI — reach for it when your priority is a local, Ollama-centric chat UI with a lighter setup.
- AnythingLLM — reach for it when the core need is document-centric RAG workspaces rather than broad multi-provider chat.
- Jan or LM Studio — reach for these when you want a desktop app for local models instead of a self-hosted multi-user server.

## Notes

- The Code Interpreter is open-source and self-hostable, powered by ClickHouse/code-interpreter, so sandboxed execution can run without sending code to a third party.
- The UI is translated into a large number of languages, and the project maintains a separate RAG API repository and website repository outside the main codebase.

## Tags

typescript, javascript, chat-ui, llm, self-hosted, web-application, agents, mcp, rag, code-interpreter, monorepo, docker
