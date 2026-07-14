# FoundationAgents/MetaGPT

> A multi-agent framework that assigns software-company roles to LLMs and runs them through a fixed SOP to turn a one-line requirement into a code repo.

[GitHub repo](https://github.com/FoundationAgents/MetaGPT) ·
[Homepage](https://atoms.dev/) ·
[License: MIT](https://github.com/FoundationAgents/MetaGPT/blob/main/LICENSE)

## Overview

MetaGPT is a Python multi-agent framework from DeepWisdom, first released by Sirui Hong (geekan) in mid-2023[^1]. Its organizing idea is `Code = SOP(Team)`: instead of letting agents freely converse toward a goal, MetaGPT hard-codes the standard operating procedure of a software company and assigns each step to a specialized LLM role — Product Manager, Architect, Project Manager, Engineer, QA Engineer. Given a single sentence ("Create a 2048 game"), it emits user stories, a competitive analysis, requirement docs, data structures, API designs, and finally source files, mimicking a waterfall pipeline[^2]. The repo redirected from `geekan/MetaGPT` to the `FoundationAgents` org; older links still resolve.

The framework attached itself to two things at once: a research line (the ICLR 2024 paper co-authored with Jürgen Schmidhuber[^3], plus later Data Interpreter, AFlow, and SPO papers) and a commercial product (MGX, "MetaGPT X", launched February 2025 as a hosted agent dev team[^4]). This split is the defining tension. The open-source repo is where the ideas are demonstrated, but the sustained product investment has moved to the hosted MGX offering — and it shows in the cadence: with ~69k stars the project is among the most-starred agent frameworks, yet the last push to `main` was January 2026, so the OSS repo is now more reference implementation than fast-moving product.

The second tension is prescription vs. flexibility. MetaGPT's SOP produces clean, legible artifacts for well-shaped toy and demo tasks, but the same rigidity that makes small runs look impressive is what limits it on messy, evolving, or large codebases where a linear PM→Architect→Engineer flow doesn't match reality.

## Getting Started

Requires Python 3.9–3.11 (not 3.12+ historically)[^5]. Node and pnpm are needed for some outputs (e.g. Mermaid diagram rendering).

```bash
pip install --upgrade metagpt
metagpt --init-config   # writes ~/.metagpt/config2.yaml
```

Edit `~/.metagpt/config2.yaml` with your LLM provider and key:

```yaml
llm:
  api_type: "openai"          # or azure / ollama / groq / anthropic etc.
  model: "gpt-4-turbo"
  base_url: "https://api.openai.com/v1"
  api_key: "YOUR_API_KEY"
```

Run from the CLI, or drive it as a library:

```python
from metagpt.software_company import generate_repo
from metagpt.utils.project_repo import ProjectRepo

repo: ProjectRepo = generate_repo("Create a 2048 game")  # writes to ./workspace
print(repo)
```

## Architecture / How It Works

The core abstraction is the **Role**. Each role has a profile, a set of **Actions**, and a private memory; roles communicate by publishing to and subscribing from a shared **Environment** message bus rather than calling each other directly (a publish/subscribe pattern, not free-form chat).

- **Team / Environment** — a `Team` owns an `Environment` that holds all roles and routes messages. `team.run(n_round=...)` steps the simulation; each round every role observes the message pool, decides whether a message is addressed to it (`_watch`), and if so runs its next Action.
- **Roles** — the default software company is `ProductManager → Architect → ProjectManager → Engineer → QAEngineer`. Each consumes the previous role's output as its input document, which is how the waterfall is enforced: the Architect can't run until a PRD message exists.
- **Actions** — the unit of LLM work. An Action wraps a prompt template, the call, and structured post-processing into an artifact (PRD, system design, task list, code file).
- **Memory** — roles keep a per-role memory of observed messages; there is a long-term memory option, but the default is short-term within a run.
- **Data Interpreter** — a separate `DataInterpreter` role that plans, writes, and executes code in a live namespace for data-analysis and ML tasks, closer to a ReAct/plan-and-execute loop than the fixed SOP[^6].

Configuration is global, not per-project: a single `~/.metagpt/config2.yaml` drives model choice, provider, and cost. The framework is provider-agnostic through an `LLMType` abstraction (OpenAI, Azure, Anthropic, Ollama, Groq, and others), so the same run can target a local model or a hosted API.

## Production Notes

**"Production" is generous.** MetaGPT is best understood as a code-generation demo engine and a research substrate, not infrastructure you run under load. Real caveats:

- **Cost per run.** A full software-company pass fans out into many sequential LLM calls (analysis, design, task breakdown, per-file generation, review). On GPT-4-class models a single non-trivial `generate_repo` run can cost real money and take minutes; there is no cheap path for iterating on large specs.
- **Output needs human fixup.** Generated repos compile and run for toy scopes (games, CRUD demos, scripted utilities). Beyond that, the linear SOP produces plausible-looking but incomplete code — the QA role catches some issues but does not substitute for a human reviewer. Treat output as a scaffold, not a deliverable.
- **Rigidity.** The SOP is the product. If your task doesn't map onto PM→Architect→Engineer (research code, exploratory data work, incremental changes to an existing large repo), you're fighting the framework. Data Interpreter exists precisely because the main pipeline is a poor fit for open-ended analysis.
- **Python version pin.** The 3.9–3.11 window (excluding 3.12+) has been a recurring packaging friction point; check the current `pyproject.toml` before assuming your interpreter works.
- **Global config footgun.** Because config lives at `~/.metagpt/config2.yaml`, running multiple projects with different models/keys means juggling one shared file or overriding config in code.
- **Node/pnpm dependency.** Easy to miss; some artifact rendering silently degrades without it.
- **Research-code surface.** The `examples/` tree carries paper implementations (AFlow, SPO, and others) of varying maturity; these are demonstrations, not supported APIs.

## When to Use / When Not

**Use when:**
- You want a one-prompt-to-repo demo of agent collaboration, or a teaching example of role-based orchestration.
- You're doing data analysis / ML scripting and want an execute-in-the-loop agent (Data Interpreter).
- You're reproducing or building on the MetaGPT / AFlow / SPO research.

**Avoid when:**
- You need fine-grained control over agent flow — the SOP is opinionated and hard to reshape.
- You're modifying a large existing codebase rather than generating a new small one.
- You need predictable cost or latency in a serving path.
- You want a maintained, fast-moving OSS product; active product work has shifted to the hosted MGX.

## Alternatives

- microsoft/autogen — use instead when you want free-form conversational multi-agent orchestration with programmable termination, rather than a fixed software-company SOP.
- crewAIInc/crewAI — use when you want lighter role/task orchestration with less ceremony and no waterfall assumption.
- langchain-ai/langgraph — use when you need to define the agent graph and state transitions yourself at a lower level.
- Significant-Gravitas/AutoGPT — use when you want a single autonomous goal-seeking agent instead of a structured team.
- openai/swarm — use when you want a minimal handoff-based multi-agent primitive with almost no framework.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| Initial public release | 2023-06 | `geekan/MetaGPT` opened; `Code = SOP(Team)` software-company pipeline[^1]. |
| Viral growth | 2023-08 | Reached the top of GitHub trending; crossed tens of thousands of stars within months. |
| ICLR 2024 paper | 2024-01 | "MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework" accepted[^3]. |
| Data Interpreter | 2024 | Execute-in-the-loop role for data/ML tasks[^6]. |
| MGX launch | 2025-02 | Hosted "MetaGPT X" agent dev team announced at mgx.dev[^4]. |
| AFlow / SPO | 2025 | AFlow accepted to ICLR 2025 (oral); SPO and AOT papers with code in `examples/`[^7]. |
| Last OSS push | 2026-01 | `main` last updated; ~69k stars, ~8.8k forks, 134 open issues at time of writing. |

## References

[^1]: MetaGPT repository, geekan (Sirui Hong) — created 2023-06-30. https://github.com/FoundationAgents/MetaGPT
[^2]: MetaGPT README, "Software Company as Multi-Agent System". https://github.com/FoundationAgents/MetaGPT#software-company-as-multi-agent-system
[^3]: Hong et al., "MetaGPT: Meta Programming for A Multi-Agent Collaborative Framework", ICLR 2024. https://openreview.net/forum?id=VtmBAGCN7o
[^4]: MGX (MetaGPT X) launch announcement — 2025-02-19. https://mgx.dev/
[^5]: MetaGPT installation guide (Python 3.9–<3.12). https://docs.deepwisdom.ai/main/en/guide/get_started/installation.html
[^6]: Data Interpreter example. https://github.com/FoundationAgents/MetaGPT/tree/main/examples/di
[^7]: "AFlow: Automating Agentic Workflow Generation", ICLR 2025. https://openreview.net/forum?id=z5uVAKwmjf

## Tags

python, multi-agent, llm, agent-framework, code-generation, gpt, autonomous-agents, orchestration, ai-agents, sop
