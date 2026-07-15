# 567-labs/instructor

> Get typed, validated objects out of any LLM by handing a Pydantic model to a patched client — one extraction API across a dozen providers.

[GitHub repo](https://github.com/567-labs/instructor) ·
[Official website](https://python.useinstructor.com/) ·
[License: MIT](https://github.com/567-labs/instructor/blob/main/LICENSE)

## Overview

Instructor is a thin library for coaxing structured output out of large language models. You declare the shape you want as a Pydantic model, pass it as `response_model` to an otherwise-normal chat completion call, and get back a validated instance of that model instead of a string. It was created by Jason Liu (`jxnl`) in mid-2023[^1] and now lives under the `567-labs` organization; the original `jxnl/instructor` path still redirects. At ~13.5k stars and pushed daily as of mid-2026, it is one of the most-depended-on pieces of glue in the Python LLM stack.

The library's defining choice is that it does almost nothing itself. It does not run its own inference, define a prompt DSL, or constrain decoding at the token level. Instead it patches the provider's own SDK client, translates your Pydantic model into whatever native structured-output mechanism that provider exposes (tool/function calling, JSON mode, or strict JSON-schema), and validates the response with Pydantic — retrying on failure by feeding the validation error back to the model. This makes it small, debuggable, and portable, but also means its guarantees are only as strong as the provider's structured-output support plus a validation-and-retry loop.

The tension worth understanding up front: Instructor is deliberately not an agent framework. There is no tool loop, no memory, no orchestration. The project's own README now routes agent-shaped use cases to PydanticAI[^2]. Instructor's value is narrow and real — reliable extraction with minimal surface area — and it is easy to over-reach it into territory it was never meant to cover.

## Getting Started

```bash
pip install instructor
# or: uv add instructor / poetry add instructor
```

```python
import instructor
from pydantic import BaseModel


class User(BaseModel):
    name: str
    age: int


client = instructor.from_provider("openai/gpt-4o-mini")

user = client.chat.completions.create(
    response_model=User,
    messages=[{"role": "user", "content": "John is 25 years old"}],
    max_retries=3,
)

print(user)  # User(name='John', age=25)
```

The same code targets other providers by changing the string: `anthropic/claude-3-5-sonnet`, `google/gemini-pro`, `ollama/llama3.2`, `groq/...`. Under the hood each resolves to that provider's SDK client and an appropriate extraction `Mode`.

## Architecture / How It Works

Instructor is a patch layer, not a runtime. The core flow:

1. **Client patching.** `from_provider(...)` (or the older `from_openai` / `from_anthropic` / the legacy `instructor.patch()`) wraps the provider SDK's `chat.completions.create` so it accepts `response_model`, `max_retries`, and related kwargs. The underlying client is otherwise untouched — you keep the provider's auth, base URL, and streaming machinery.
2. **Schema translation via Mode.** A `Mode` enum selects how the Pydantic schema is delivered to the model: `TOOLS`/`FUNCTIONS` (function-calling), `JSON`/`MD_JSON` (JSON in the message body), or strict variants (`TOOLS_STRICT`) that lean on OpenAI-style guaranteed JSON-schema decoding. The right mode is provider-dependent, and picking a weaker mode silently degrades reliability.
3. **Validation + retry.** The raw completion is parsed into your model. If Pydantic validation fails, Instructor re-sends the request with the validation error appended, up to `max_retries` (backed by Tenacity). Exhausting retries raises `InstructorRetryException`, which carries the last error and the failed completions.
4. **Streaming.** `Partial[Model]` yields progressively-filled instances as tokens arrive, and `Iterable[Model]` streams a sequence of objects, using a partial-JSON parser. Validators do **not** run on partial objects — only the final object is fully validated.

The coupling story is the whole story: Instructor's correctness is a composition of (a) how well the chosen provider mode constrains output and (b) how many retries you allow. On providers with true constrained decoding, near-100% schema adherence is achievable in one shot; on JSON-mode-only providers, you are relying on the retry loop to converge. Instructor makes this uniform to call but does not make it uniform in cost or reliability.

## Production Notes

- **Retries multiply spend and latency.** Each retry is a full round trip with the schema and prior error re-sent. A model that reliably fails one field can silently triple your token bill and wall-clock time before raising. Set `max_retries` deliberately, log `InstructorRetryException`, and alert on retry rates — a rising retry rate is usually a prompt or schema regression, not noise.
- **Schema is prompt overhead.** The model schema is injected into every request (as a tool definition or inline JSON schema). Large, deeply-nested models add meaningful input tokens on every call, including retries.
- **Strict mode is not universal.** OpenAI-style strict JSON-schema decoding rejects many valid Pydantic constructs (certain unions, unbounded dicts, some default/optional patterns). A model that validates in Python may not be expressible as a strict schema; Instructor falls back to a looser mode, changing your reliability profile without an obvious signal.
- **Streaming skips validators.** Because `field_validator`/`model_validator` logic runs only on the completed object, you cannot trust constraints on `Partial[...]` output mid-stream. Treat partial objects as UI hints, not validated data.
- **API churn across versions.** The public surface has moved from `instructor.patch()` → `from_openai()`/`from_anthropic()` → the unified `from_provider()`. Much online tutorial code targets deprecated entry points. Pin the version and follow the current docs, not blog posts.
- **Pydantic v2 only.** Instructor requires Pydantic v2; v1 is unsupported. This is usually a non-issue now but bites older codebases mid-migration.
- **It is not observability or eval infrastructure.** There is no built-in tracing, dataset replay, or dashboarding. Teams that outgrow raw extraction typically add their own logging or move to a framework that bundles it.

## When to Use / When Not

**Use when:**
- You need clean, validated Python objects from LLM output and want to keep the provider SDK you already use.
- You want one extraction API that ports across OpenAI, Anthropic, Google, local models, and others.
- Your task is bounded extraction/classification/parsing, not multi-step agent behavior.
- You value a small, readable dependency you can debug by reading its source.

**Avoid when:**
- You need agents: tool loops, memory, planning, observability. Reach for a framework instead.
- You require hard, token-level output guarantees on local/open models — constrained-decoding libraries enforce the grammar during generation rather than validating after.
- Your budget can't absorb a retry loop and you have no way to cap or monitor it.
- You are on Pydantic v1 and cannot migrate.

## Alternatives

- pydantic/pydantic-ai — the Pydantic team's agent framework; use instead when you need typed tools, evals, and tracing, not just extraction. Instructor's own README points here for agent use cases.
- dottxt-ai/outlines — constrained decoding for local/open models; use instead when you need grammar-level guarantees during generation rather than validate-and-retry.
- guidance-ai/guidance — constrained generation and templating; use instead when you want fine-grained control over the generation structure itself.
- boundaryml/baml — a dedicated schema DSL with its own type system and tooling; use instead when you want structured outputs defined outside Python with generated clients.
- prefecthq/marvin — higher-level AI functions/utilities; use instead when you want ergonomic task helpers rather than a bare extraction primitive.

## History

Version dates below are approximate where noted; treat the milestones, not the exact days, as authoritative.

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-06 | First release: `patch()` over the OpenAI client, `response_model` via function calling[^1]. |
| 1.0 | ~2024 | API redesign: `from_openai`/`from_anthropic`, `Mode` enum, Tenacity-backed retries, multi-provider support. |
| — | ~2024–2025 | Streaming via `Partial`/`Iterable`, strict JSON-schema modes, sibling ports (TypeScript, Ruby, Go, Elixir, Rust). |
| — | ~2025 | Unified `from_provider("provider/model")` entry point becomes the recommended API. |

## References

[^1]: Instructor documentation and repository history, 567-labs/instructor (repo created 2023-06-14). https://python.useinstructor.com/
[^2]: Instructor README, "Use Instructor for fast extraction, reach for PydanticAI when you need agents." https://github.com/567-labs/instructor

## Tags

python, pydantic, structured-outputs, llm, openai, anthropic, json-schema, validation, extraction, data-extraction
