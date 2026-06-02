# earendil-works/pi

An AI agent toolkit positioned as "coding agent CLI + unified LLM API + TUI/web UI libraries + Slack bot + vLLM pods". Audit signals warrant a hands-on review before integration.

## What it is

A TypeScript-implemented bundle that combines a coding-agent CLI, a unified abstraction over multiple LLM providers, TUI and web UI libraries, Slack integration, and vLLM-pod orchestration. README description is concise; the repo's distinguishing characteristic in the cold-start corpus is anomalous star velocity relative to its watcher count and project age.

## Key features

- Coding-agent CLI surface.
- Unified LLM API across providers.
- TUI + web UI libraries shipped from the same monorepo.
- Slack bot connector.
- vLLM-pod orchestration (the "pods" framing maps to GPU-served inference).
- MIT-licensed.

## Tech stack

- TypeScript primary.

## When to reach for it

- You're cataloguing the broader agent-toolkit space.
- You're explicitly attracted to the "everything in one repo" framing and willing to audit the code.

## When *not* to reach for it

- You want vendor-supported tooling — first-party Anthropic, OpenAI, or Microsoft offerings are safer.
- You're risk-averse about installing third-party CLI / Slack / inference code into credentialed environments without review.

## Maturity signal

59k stars vs. 206 watchers + no topic tags + minimal README description is unusual for an organically-grown OSS project. None of these signals is conclusive, but together they place this firmly in the "verify the actual code and the maintainer history before depending" bucket. MIT license clean.

## Alternatives

- `anthropics/claude-code`, `cline/cline` — first-party + well-known coding agents.
- `obra/superpowers`, `affaan-m/ECC` — third-party agent / skills frameworks.
- LangChain / DSPy directly — for code-first LLM-app composition.

## Tags

artificial-intelligence, large-language-model, agent, typescript, command-line-interface, framework
