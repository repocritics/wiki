# github/github-mcp-server

GitHub's official Model Context Protocol server that connects AI agents to GitHub repositories, issues, pull requests, and workflows.

## What it is

The GitHub MCP Server exposes GitHub's platform to MCP-compatible AI tools so that assistants and agents can read code, manage issues and pull requests, inspect CI runs, and review security findings through natural-language interactions. It is aimed at developers who want to give their AI host (VS Code, Claude, Cursor, and similar) grounded access to GitHub context and actions. It ships as both a GitHub-hosted remote server and a self-run local server.

## Key features

- Two deployment paths: a remote server hosted at `api.githubcopilot.com/mcp/` and a local server run via Docker image or a `go build` binary in stdio mode.
- Tools grouped into toolsets (context, repos, issues, pull_requests, actions, code_security, and many more), selectable with `--toolsets`/`GITHUB_TOOLSETS` or individually with `--tools`.
- OAuth login (kept in memory) or Personal Access Token authentication, with the PAT taking precedence when set.
- A read-only mode that skips all write tools and takes priority even over explicitly requested tools.
- GitHub Enterprise support via `--gh-host`/`GITHUB_HOST`, plus an insiders mode for early-access tools.
- A `tool-search` CLI subcommand for discovering tools by name, description, or parameter.

## Tech stack

- Go, with the module targeting Go 1.25.0.
- `google/go-github/v89` for the REST API and `shurcooL/githubv4` plus `shurcooL/graphql` for GraphQL.
- The official `modelcontextprotocol/go-sdk` at v1.7.0-pre.3.
- Cobra, pflag, and Viper for CLI and configuration; go-chi for HTTP routing.
- `golang.org/x/oauth2` for auth and `microcosm-cc/bluemonday` for HTML sanitization.
- Distributed as a public Docker image at `ghcr.io/github/github-mcp-server`.

## When to reach for it

- Giving an AI coding assistant direct, authenticated access to your GitHub repositories and metadata.
- Automating issue and pull-request triage, review, and project maintenance through an agent.
- Monitoring GitHub Actions runs and analyzing build failures from within an AI host.
- Surfacing code scanning, secret scanning, and Dependabot alerts to an assistant for analysis.

## When *not* to reach for it

- Deterministic, scripted automation where a fixed API client or the GitHub CLI is more predictable than an LLM.
- Hosts that do not support MCP, which cannot use the server at all.
- GitHub Enterprise Server, which the remote server does not host — those users must run the local server with their own OAuth or GitHub App.
- Granting broad token scopes to an AI tool when your security posture cannot tolerate it.

## Maturity signal

Created in early 2025, the project already has over 31,000 stars, an MIT license, and commits pushed within the last day, marking it as actively maintained and rapidly evolving. The presence of remote and local paths, extensive per-tool snapshot tests, and insiders/feature-flag machinery point to production-grade engineering backed directly by GitHub. Being first-party lowers the risk of it lagging behind GitHub API changes.

## Alternatives

- GitHub CLI (`gh`) — use instead when you want deterministic, scriptable automation rather than agent-driven natural-language access.
- GitLab MCP Server — use instead when your forge is GitLab rather than GitHub.
- go-github or Octokit directly — use instead when building a fixed programmatic integration that does not involve an LLM agent.

## Notes

Read-only mode is enforced as a hard priority: even if a write tool is explicitly named via `--tools`, it is skipped when `--read-only` is set. The server also preserves old tool names as aliases when tools are renamed, so upgrades do not silently break existing agent prompts.

## Tags

golang, mcp, mcp-server, github, developer-tools, devops, ai-agents, cli, api-client, automation
