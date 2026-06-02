# Significant-Gravitas/AutoGPT

An autonomous-agent platform that pioneered the "GPT loops on itself to complete tasks" pattern, since transformed into a self-host-or-cloud platform for building, deploying, and running AI agents.

## What it is

AutoGPT started in March 2023 as the viral demo of LLM-driven autonomous task execution — give it a goal and watch it plan, take actions, and iterate. The codebase has since matured from "single Python script" into a platform with a server, a no-code agent builder, marketplace components, and supporting infrastructure. The README's current framing is "build, deploy, and run AI agents" — the autonomous-loop primitive that put the project on the map is now one of several execution modes.

## Key features

- Agent platform with both a server runtime and a visual builder (no-code agent construction).
- LLM-provider flexibility — works with OpenAI, Anthropic Claude, Llama-family hosted endpoints.
- Marketplace component for sharing pre-built agents.
- Discord community + Twitter presence as the primary support surfaces.
- Multilingual README translation tracks (German, Spanish, and others via the zdoc.app translation network).

## Tech stack

- Python primary across the agent runtime.
- Web-based no-code builder ships as part of the platform.
- Docker-first deployment for self-hosting.

## When to reach for it

- You want a polished agent platform with a UI rather than building agent infrastructure yourself.
- You're experimenting with autonomous-agent patterns and want a reference implementation that's actively maintained.
- You're studying the canonical example of the early-2023 "auto-agent" wave and where it evolved.

## When *not* to reach for it

- You want a lightweight, code-only agent framework — LangChain, LlamaIndex, or DSPy fit that need.
- You're allergic to non-OSI licenses — the SPDX is `NOASSERTION`; check the LICENSE file before commercial deployment.
- You need vendor-grade SLAs — even the cloud offering is a self-service product, not an enterprise contract.

## Maturity signal

184k stars, 46k forks, last push the day before this page was generated. 3-year-old project that survived the autonomous-agent hype cycle of 2023 to become a sustained platform play. The 437 open-issues count is moderate for the platform's surface area; community velocity has slowed from the 2023 peak but the project is in active development under the Significant Gravitas organization. License `NOASSERTION` requires direct LICENSE-file inspection before commercial assumptions.

## Alternatives

- LangChain / LangGraph — use when you want a Python library for agent composition rather than a platform.
- CrewAI, Autogen, ChatDev — use when you want role-based multi-agent orchestration.
- `obra/superpowers`, `affaan-m/ECC`, plain CLAUDE.md — use when you're scaffolding skills for an existing CLI coding agent rather than running an agent server.

## Notes

The project's 2023-vintage README iconography (Discord/Twitter buttons leading the README, marketing translations via zdoc.app) reflects its origins as a viral demo before the platform pivot. The marketplace and no-code builder shifted the value proposition from "the loop" to "the platform" — anyone evaluating today should weigh AutoGPT against agent platforms (n8n, Make.com, Activepieces) more than against agent libraries (LangChain, DSPy). License uncertainty is the recurring blocker for enterprise pilots; verify before committing.

## Tags

artificial-intelligence, large-language-model, agent, autonomous-agents, python, platform, claude, openai, no-code
