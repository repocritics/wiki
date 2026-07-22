# cft0808/edict

A multi-agent orchestration system built on OpenClaw that models AI collaboration on the Tang dynasty's "Three Departments and Six Ministries" bureaucracy, adding a mandatory review stage and a real-time dashboard.

## What it is

Edict (三省六部) coordinates twelve specialized AI agents through a fixed, government-inspired workflow: a triage agent sorts incoming messages, a planning department decomposes tasks, a review department approves or rejects plans, a dispatch department assigns work, and several ministry agents execute in parallel. It is aimed at people who want multi-agent task execution that is auditable, observable, and interruptible rather than a black box. Compared with frameworks like CrewAI or AutoGen, its distinguishing pieces are an institutional review gate that can reject and force rework, and a real-time kanban dashboard. It requires OpenClaw to run the agents.

## Key features

- A twelve-agent hierarchy with an explicit permission matrix governing which agent may message which, plus state-transition validation that rejects illegal task status jumps.
- A mandatory review department (门下省) that inspects planning output and can reject it back for rework before execution proceeds.
- A real-time kanban dashboard with ten panels: task board, dispatch monitor, archived "memorials" with a five-stage timeline, template library, officials/token-usage overview, news feed, per-agent model config, skills management, session monitoring, and a daily ceremony animation.
- Per-agent configuration: each agent has its own workspace, skills, and hot-swappable LLM (changing a model restarts the gateway to take effect).
- Remote skills management from GitHub/URLs via the dashboard UI, a CLI (`skill_manager.py`), or a REST API.
- An async backend with a Redis Streams EventBus, a transactional Outbox Relay for at-least-once event delivery, a DAG orchestrator, and a parallel dispatch worker with exponential-backoff retry and resource locks; all state changes are written to an audit log.

## Tech stack

- Python 3.10+ (the README badge shows 3.9+; the install prerequisites state 3.10+); macOS or Linux.
- A dashboard server built on the Python standard library (`http.server`) with zero required dependencies, serving both the API and static files; optional numpy for the LinUCB model-recommendation router and optional Playwright for screenshots.
- A React 18 frontend with TypeScript, Vite, and Zustand (13 components), plus a single-file zero-dependency `dashboard.html` fallback.
- An async backend service using SQLAlchemy and Redis (Alembic migrations present under `edict/backend`).
- Runs on OpenClaw (required); ships install.sh/start.sh scripts, a systemd unit, and a Docker demo image with prebuilt frontend and mock data.
- MIT licensed.

## When to reach for it

- Running multi-agent tasks where you need a mandatory quality-review/approval step before work is executed.
- Situations that demand full observability and intervention — pausing, cancelling, or resuming tasks and inspecting the complete flow of each one.
- Users already running OpenClaw who want a dashboard-driven orchestration layer on top of it.
- Trying the concept quickly via the Docker demo image with preloaded sample data.

## When *not* to reach for it

- The full system requires OpenClaw and API keys configured per agent; if you are not on OpenClaw it does not apply (only the demo dashboard runs standalone).
- The rigid department workflow with a forced review gate adds latency and structure that is unnecessary for simple single-agent or one-shot tasks.
- The heavy metaphor (imperial roles, Chinese-first docs) may be a barrier for teams wanting a neutral, plainly named framework.
- The full async backend brings Redis and a database into the picture; overkill when you only need lightweight coordination.

## Maturity signal

Created in February 2026 and pushed within days of this writing, the project is actively developed and has attracted a large audience. The Phase 1 roadmap is largely complete (twelve-agent architecture, dashboard, memorials, templates, tests), while Phase 2 shows a mix of shipped infrastructure (EventBus, Outbox Relay, state-machine audit, DAG orchestrator) and still-open items such as a human-approval mode. End-to-end tests (17 assertions) and a state-machine consistency test exist. It reads as a young but energetically maintained project whose core is stable and whose institutional features are still deepening.

## Alternatives

- CrewAI — use it when you want a widely adopted role-based multi-agent framework and do not need a built-in review gate or dashboard.
- AutoGen — use it when you want conversational multi-agent orchestration with human-in-the-loop, without the fixed department pipeline.
- MetaGPT — use it when you want a company-metaphor multi-agent workflow for software tasks rather than the imperial-bureaucracy model here.
- LangGraph — use it when you want to build custom agent graphs and state machines yourself instead of adopting a prescribed hierarchy.

## Notes

- The dashboard server is deliberately dependency-free: `server.py` (~2300 lines) runs on `http.server` and serves both the API and the prebuilt React frontend, and a single-file `dashboard.html` (~2500 lines) works with no build step.
- State transitions are enforced by a `_VALID_TRANSITIONS` state machine in `kanban_update.py`; illegal jumps (for example Doing to Taizi) are rejected and logged, and every change is appended to an audit log.
- Incoming task titles and notes are automatically cleaned of file paths, metadata, and invalid prefixes before a task is created.

## Tags

python, react, multi-agent, ai-orchestration, llm, dashboard, kanban, workflow-automation, redis, event-driven
