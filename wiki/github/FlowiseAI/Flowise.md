# FlowiseAI/Flowise

A visual drag-and-drop builder for LLM workflows — direct competitor to LangFlow, with overlapping target audience.

## What it is

A TypeScript + React visual editor for composing LLM workflows: connect nodes for LLM calls, prompts, retrievers, agents, tools, memory. Generates executable workflows that can be served as REST endpoints or embedded as chat interfaces. Built on top of LangChain.js. Aimed at engineers + non-engineers collaborating on agent / RAG pipelines.

## Key features

- Drag-and-drop visual editor for LLM workflows.
- Pre-built node templates for common patterns (RAG, agents, chatbots).
- Embeddable as a chat widget on external sites.
- Self-host (Docker, Node) or use Flowise Cloud.
- Built on LangChain.js as the underlying composition layer.
- License is `NOASSERTION` — verify LICENSE before commercial deployment.

## Tech stack

- TypeScript primary.
- React + React Flow for the visual editor.
- LangChain.js as the runtime substrate.
- Node.js backend.

## When to reach for it

- You want a visual LLM-workflow builder and find Flowise's UX closer-fit than LangFlow.
- You're shipping internal LLM tools where non-coders need to compose workflows.
- You want an embeddable chat widget out of the box.

## When *not* to reach for it

- You're code-first — LangChain.js directly is lighter.
- You're allergic to non-OSI licenses — `NOASSERTION` carries commercial restrictions; verify.
- You want vendor-stable observability — Flowise's metrics surface is thinner than commercial alternatives.

## Maturity signal

53k stars, 24k forks, last push recent. The 24k fork count reflects self-hosting deployments. License absence is the recurring caveat.

## Alternatives

- `langflow-ai/langflow` — Python-flavored direct competitor.
- `n8n-io/n8n` — broader workflow automation, AI as one node type.
- LangChain / LangGraph directly — code-first.

## Tags

artificial-intelligence, large-language-model, agent, typescript, react, langchain, workflow-automation, low-code, self-hosted
