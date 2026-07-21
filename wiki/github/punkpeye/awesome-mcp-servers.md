# punkpeye/awesome-mcp-servers

A community-maintained catalog of Model Context Protocol (MCP) servers.

## What it is

This is an awesome-list style directory that indexes MCP servers — the components that expose tools and data sources to LLM clients through the Model Context Protocol. It exists so that developers building with agents and LLM tooling can discover existing servers rather than searching from scratch. The repository is documentation, not runnable software: the value is in the curated index and its contributor process.

## Key features

- Curated index of MCP servers maintained through community contributions, with a `CONTRIBUTING.md` describing how to submit entries.
- Internationalized: the catalog ships parallel READMEs in Korean, Japanese, Chinese (Simplified and Traditional), Portuguese (Brazil), Thai, and Persian, alongside the English `README.md`.
- Paired with the Glama MCP directory: the project homepage points to `glama.ai/mcp/servers`, and a `check-glama.yml` GitHub Actions workflow keeps the list in sync with that directory.
- MIT licensed, so entries and structure can be reused freely.

## Tech stack

- Content is Markdown; there is no build system or package manifest in the repository.
- Continuous integration is a single GitHub Actions workflow (`.github/workflows/check-glama.yml`) tied to the Glama directory.
- The catalog itself indexes MCP servers; the specific servers, languages, and runtimes covered live in the README content, which is not included in these inputs.

## When to reach for it

- Looking for an existing MCP server for a particular data source, API, or tool before writing your own.
- Surveying the breadth of the MCP ecosystem to understand what integrations already exist.
- Finding reference points when contributing or publishing a new MCP server.

## When *not* to reach for it

- You need the protocol specification or SDKs — this is a directory of implementations, not the standard itself.
- You need a runnable server or library — nothing here executes; each entry links out to a separate project.
- You need vetted, production-guaranteed integrations — an awesome list is curated, not audited, and the large open-issue count reflects ongoing submission churn.
- You want programmatic search or filtering over server metadata — the linked Glama directory is the machine-friendly surface.

## Maturity signal

Actively maintained and heavily followed. With roughly 91,000 stars and 13,000 forks against a creation date of late November 2024 — essentially the moment MCP entered public use — this is among the most-followed community catalogs in the space, and the star trajectory reflects how quickly the ecosystem grew. The last push (2026-07-13) is recent, the repository is not archived, and the roughly 2,900 open issues are consistent with a popular awesome list absorbing a steady stream of submission requests rather than a sign of neglect.

## Alternatives

- `modelcontextprotocol/servers` — use the official reference-server collection when you want first-party implementations and canonical examples.
- Glama MCP directory (`glama.ai/mcp/servers`) — use the web directory this repo feeds when you want searchable, filterable metadata instead of a flat Markdown list.
- `mcp.so` — use a hosted registry when you prefer a browsable web catalog with categories over a GitHub-hosted list.

## Notes

- The repository is coupled to an external directory rather than standing alone: the homepage and the `check-glama.yml` workflow both point at Glama, so the list is validated against that service rather than maintained purely by hand.
- The eight-language README set signals a deliberately international audience, unusual for a developer awesome list of this kind.

## Tags

awesome-list, mcp, model-context-protocol, ai, llm, agents, directory, catalog, documentation, markdown
