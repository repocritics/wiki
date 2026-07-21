# multica-ai/multica

An open-source, self-hostable platform for managing coding agents as teammates — assigning them issues, tracking progress, and reusing skills across a shared board.

## What it is

Multica is a managed-agents platform that treats coding agents like colleagues on a project board. You assign an issue to an agent the way you would to a person, and it picks up the work, writes code, reports blockers, and updates its status autonomously. It is aimed at human-plus-AI teams that want vendor-neutral, self-hosted infrastructure for coordinating multiple agents rather than manually copy-pasting prompts into individual CLIs.

## Key features

- Agents as teammates: agents have profiles, appear on the board, post comments, create issues, and report blockers, with a full task lifecycle (enqueue, claim, start, complete/fail).
- Squads: group agents and humans under a leader agent and assign work to the squad, so the leader routes tasks and assignment stays stable as the team grows.
- Autopilots: schedule recurring agent work via cron triggers, webhooks, or manual runs, each creating an issue and routing it automatically.
- Reusable skills that let solutions (deployments, migrations, code reviews) compound into shared team capabilities.
- Unified runtimes with one dashboard for local daemons and cloud runtimes, auto-detecting available agent CLIs, plus real-time progress streaming over WebSocket.
- Multi-workspace isolation, with each workspace having its own agents, issues, and settings.

## Tech stack

- Frontend: Next.js 16 (App Router); primary repository language reported as Go.
- Backend: Go using the Chi router, sqlc, and gorilla/websocket; requires Go v1.26+.
- Database: PostgreSQL 17 with pgvector.
- Agent runtime: a local daemon that auto-detects and executes many agent CLIs (Claude Code, Codex, CodeBuddy, GitHub Copilot CLI, OpenCode, OpenClaw, Hermes, Pi, Cursor Agent, Kimi, Kiro CLI, Antigravity, Qoder CLI, Trae CLI).
- Monorepo tooling: pnpm `10.28.2`, Turbo, and TypeScript; Node.js v20+ for development, Docker for self-hosting, with Electron desktop and iOS mobile clients included.
- Repository version `0.2.0`. License: reported as NOASSERTION (non-standard/unrecognized).

## When to reach for it

- Coordinating several coding agents across a team with an issue board, assignees, and an activity timeline instead of running each agent by hand.
- Delegating work to a group and letting a leader agent route it, when direct per-agent assignment does not scale.
- Scheduling recurring agent tasks such as standups, weekly reports, or periodic audits.
- Self-hosting agent infrastructure that stays vendor-neutral across many different agent CLIs.

## When *not* to reach for it

- You only run a single agent interactively and do not need a board, routing, or lifecycle management.
- You cannot provision the required stack (Go, PostgreSQL with pgvector, Docker) or prefer a fully hosted product with no infrastructure to run.
- Your team has no established issue-driven workflow, so the task-lifecycle model adds overhead without benefit.

## Maturity signal

Multica is a young project, created in early 2026, that has attracted a very large star count and shows a push date within the last day — signs of rapid, active development and early attention. A high open-issue count is consistent with a fast-moving platform still stabilizing rather than one in maintenance. The most important caveat is licensing: the license resolves to NOASSERTION, meaning it is non-standard or unrecognized, so review the actual license terms before relying on it commercially.

## Alternatives

- OpenHands — reach for it when you want an autonomous software-agent runtime rather than a team board and routing layer over existing CLIs.
- Claude Code or Codex CLI used directly — reach for these when a single interactive agent is enough and you do not need multi-agent management.
- An existing issue tracker (for example Linear or Jira) with manual agent runs — reach for that when you already have project management and only occasionally hand work to an agent.

## Notes

- The name is a nod to Multics, the 1960s time-sharing operating system, extending its multiplexing idea to teams where both humans and agents share the system.
- Beyond the web app, the repository ships a desktop client (Electron) and an iOS mobile client, and self-hosting pulls official images from GHCR with a `make selfhost-build` fallback when a tag is unpublished.

## Tags

go, typescript, nextjs, postgresql, ai-agents, orchestration, self-hosted, websocket, devops, developer-tools, monorepo, daemon
