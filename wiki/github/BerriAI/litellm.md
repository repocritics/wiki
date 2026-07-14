# BerriAI/litellm

> A single OpenAI-format interface — as a Python SDK and a self-hosted proxy — for calling 100+ LLM providers.

[GitHub repo](https://github.com/BerriAI/litellm) ·
[Official website](https://docs.litellm.ai/docs/) ·
[License: MIT](https://github.com/BerriAI/litellm/blob/main/LICENSE) (the `enterprise/` directory is under a separate commercial license)

## Overview

LiteLLM is a translation layer that exposes 100+ LLM providers — OpenAI, Anthropic, Google Gemini/Vertex, AWS Bedrock, Azure OpenAI, Cohere, HuggingFace, local vLLM/Ollama, and many smaller vendors — behind one request/response shape: OpenAI's Chat Completions format. It ships in two forms from the same codebase[^1]. The **Python SDK** (`litellm`) is an in-process function call: `completion(model="anthropic/claude-...", messages=[...])` returns an OpenAI-shaped object regardless of the underlying vendor. The **Proxy Server / AI Gateway** wraps that SDK in a FastAPI service that adds virtual API keys, per-key/team budgets and spend tracking, rate limiting, load balancing across deployments, fallbacks, guardrails, caching, and logging callbacks.

It is built and maintained by BerriAI (Krrish Dholakia and Ishaan Jaffer), a Y Combinator W23 company[^2]. The project's reason to exist is that every provider has a slightly different SDK, auth pattern, request schema, streaming format, and error taxonomy; LiteLLM absorbs those differences so application code and observability tooling only ever speak one dialect. That is also its defining tension: the value is entirely in the fidelity of a large, hand-maintained per-provider translation layer, and provider APIs change constantly. The result is a fast-moving library (often multiple releases per day) where breadth of coverage is traded against surface-area bugs and a large dependency footprint.

The proxy is the part most teams standardize on: it turns "N services each holding N provider keys" into "one gateway holding the keys, issuing scoped virtual keys, and reporting cost centrally."

## Getting Started

```bash
pip install litellm          # SDK only
pip install 'litellm[proxy]' # SDK + gateway server
```

SDK — one call shape across providers:

```python
from litellm import completion
import os

os.environ["ANTHROPIC_API_KEY"] = "sk-ant-..."

resp = completion(
    model="anthropic/claude-sonnet-4-20250514",
    messages=[{"role": "user", "content": "Hello!"}],
)
print(resp.choices[0].message.content)   # OpenAI-shaped response
```

Proxy — front a fleet of models behind one endpoint. A `config.yaml`:

```yaml
model_list:
  - model_name: gpt-4o                       # the name clients request
    litellm_params:
      model: azure/my-gpt-4o-deployment
      api_base: https://example.openai.azure.com
      api_key: os.environ/AZURE_API_KEY
  - model_name: gpt-4o                       # same name = a load-balanced pool
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY
```

```bash
litellm --config config.yaml   # serves on http://0.0.0.0:4000
```

Clients then point the OpenAI SDK at the proxy and never learn the real backend:

```python
import openai
client = openai.OpenAI(api_key="sk-litellm-virtual-key", base_url="http://0.0.0.0:4000")
client.chat.completions.create(model="gpt-4o", messages=[{"role": "user", "content": "Hi"}])
```

## Architecture / How It Works

The SDK's core is a per-provider transformation registry. Each provider implements two directions: map an OpenAI-format request into the vendor's native request, and map the vendor's response (and streaming chunks) back into OpenAI's `ChatCompletion` / `ChatCompletionChunk` objects. `completion()` is the sync entry point; `acompletion()` is the async one. Exceptions from every provider are normalized into the OpenAI exception hierarchy (`RateLimitError`, `AuthenticationError`, `APIError`, etc.) so callers write one error-handling path.

Cost tracking depends on a large JSON map, `model_prices_and_context_window.json`, that records per-token input/output pricing and context limits for every known model. Usage from responses is multiplied against this map to compute spend. Because the map is hand-maintained, newly released models or repriced ones can be temporarily missing or wrong.

The **Router** is the load-balancing brain shared by SDK and proxy. Multiple `litellm_params` entries sharing one `model_name` form a deployment pool; the Router distributes across them (simple shuffle, least-busy, latency-based, or usage-based routing), applies retries with exponential backoff, cools down deployments that error, and falls back to alternate models on failure.

The **Proxy** is FastAPI in front of the Router. It adds: virtual keys and teams persisted in **Postgres** (via Prisma), budget/rate-limit enforcement, an admin UI, **guardrails** (PII, prompt-injection, moderation hooks — some via third-party integrations), a **caching** layer (in-memory or Redis, keyed on request semantics), and **logging callbacks** that fan out each request to observability backends (Langfuse, OpenTelemetry, Datadog, S3, and others). More recent surfaces extend the same idea beyond chat: an MCP gateway that exposes MCP tool servers through the proxy, A2A agent invocation, embeddings, images, audio, rerank, and pass-through routes for providers' native formats (e.g. Anthropic's `/messages`).

## Production Notes

**The proxy needs Postgres.** Virtual keys, teams, and spend live in a database via Prisma. First boot runs migrations; a mismatched schema between the DB and the running image version is a common upgrade failure. Budgetback pressure and rate limits also lean on Redis in multi-replica deployments — without a shared Redis, per-key counters and cache are per-process and drift.

**Release velocity is a footgun.** LiteLLM ships extremely frequently, and minor bumps have carried behavioral and config-schema changes. Pin an exact version (`litellm==x.y.z`), read release notes before upgrading, and test the proxy config against the target version rather than tracking latest in production.

**Translation fidelity varies by provider.** The OpenAI-in / OpenAI-out promise is strongest for the mainstream providers (OpenAI, Azure, Anthropic, Bedrock, Vertex, Gemini) and thinner for long-tail vendors. Provider-specific parameters, tool/function-calling schemas, streaming edge cases, and multimodal payloads are where per-provider bugs concentrate. Treat non-mainstream providers as "verify before you depend."

**Cost numbers are best-effort.** Spend tracking is only as accurate as `model_prices_and_context_window.json`. For finance-grade billing, reconcile LiteLLM's reported spend against each provider's own invoices rather than trusting the gateway as the source of truth.

**Overhead and scale.** The gateway adds a network hop and JSON re-serialization; documented latency is low (single-digit-millisecond P95 in the project's own benchmarks[^3]), but real throughput depends on Postgres/Redis sizing, callback fan-out cost, and worker count. Logging callbacks run in the request path unless configured asynchronously — a slow observability backend can back-pressure completions.

**Dependency weight.** The SDK pulls in a broad dependency set to cover all providers; the `[proxy]` extra adds FastAPI, Prisma, and more. In constrained environments, install only what you use and watch for transitive version conflicts.

## When to Use / When Not

**Use when:**
- You call more than one provider and want to swap models without rewriting call sites.
- You need a central gateway for API keys, budgets, spend attribution, and rate limits across a team.
- You want existing OpenAI-SDK clients and tools to reach non-OpenAI models unchanged.
- You need fallbacks and load balancing across deployments (e.g. Azure quota → OpenAI overflow).

**Avoid when:**
- You call exactly one provider and value a small dependency tree — use that vendor's official SDK.
- You need to serve/host the model weights themselves — that is an inference engine's job, not a gateway's.
- You require a locked, slow-moving dependency; LiteLLM's release cadence is high and occasionally breaking.
- You need billing-grade cost accuracy without reconciliation against provider invoices.

## Alternatives

- Portkey-AI/gateway — a fast Rust/TypeScript AI gateway; use it when you want a lightweight, low-overhead proxy and don't need LiteLLM's Python-SDK surface.
- vllm-project/vllm — a serving engine that hosts open-weight models; use it when the problem is running the model, not routing to hosted APIs (LiteLLM can then proxy vLLM).
- langchain-ai/langchain — a full orchestration framework; use it when you need chains/agents/retrieval, not just a normalized call layer.
- Helicone/helicone — observability-first proxy for logging, caching, and cost dashboards; use it when monitoring is the primary need over multi-provider translation.
- simonw/llm — a CLI and small Python library for multi-provider access; use it for scripting and local experimentation rather than a team gateway.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-07 | Repo created; SDK-first unified `completion()` interface[^2]. |
| — | 2023 | BerriAI in Y Combinator W23; proxy server / gateway added. |
| 1.x | 2024 | Router, virtual keys, budgets, Postgres-backed proxy, admin UI, broad provider growth. |
| 1.x | 2025 | Guardrails, expanded logging callbacks, responses/rerank/audio surfaces, enterprise tier. |
| 1.x | 2026 | MCP gateway, A2A agent invocation, provider-native pass-through routes; 100+ providers. |

Version numbers are intentionally coarse here: LiteLLM releases on a `1.y.z` line at very high frequency, and mapping specific features to exact patch versions is unreliable. Consult the GitHub releases for precise dates.

## References

[^1]: LiteLLM documentation — SDK and Proxy Server (AI Gateway). https://docs.litellm.ai/docs/
[^2]: BerriAI on Y Combinator. https://www.ycombinator.com/companies/berriai
[^3]: LiteLLM proxy benchmarks. https://docs.litellm.ai/docs/benchmarks

## Tags

python, llm, ai-gateway, llm-gateway, proxy, openai-compatible, llmops, load-balancing, api-gateway, mcp, cost-tracking
