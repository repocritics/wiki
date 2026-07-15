# guardrails-ai/guardrails

> A Python framework that wraps LLM calls with input/output validators to detect, quantify, and mitigate risks — and to coerce outputs into typed structures.

[GitHub repo](https://github.com/guardrails-ai/guardrails) ·
[Official website](https://www.guardrailsai.com/docs) ·
[License: Apache-2.0](https://github.com/guardrails-ai/guardrails/blob/main/LICENSE)

## Overview

Guardrails sits between your application and an LLM and runs *validators* over the model's inputs and outputs. A validator is a small unit that checks one property — "no PII", "not toxic", "matches this regex", "doesn't mention a competitor", "is valid JSON against this schema" — and declares what to do on failure: raise, filter the offending span, re-ask the model, or silently pass. Validators are composed into a `Guard`, which is the object you call instead of (or around) the raw LLM API[^1].

The project has two historical centers of gravity, and the tension between them defines it. It launched in 2023 around **RAIL** (Reliable AI markup Language), an XML dialect for specifying both an output schema and its validators in one document, with the framing that structured-output-from-LLMs was the core problem[^2]. As function calling and dedicated structured-output libraries matured, that framing weakened, and the project's center shifted to the **Guardrails Hub** — a registry of pre-built validators installed individually — plus a fluent `Guard().use(...)` API. RAIL still exists but is de-emphasized; new code uses Pydantic models or direct validator composition[^1]. If you read older tutorials you will hit RAIL XML; if you read the current docs you will not.

Guardrails is Python-first (a JavaScript client exists for the server). It is maintained by the company Guardrails AI, which also runs the hosted Hub and publishes the Guardrails Index, a benchmark comparing validator accuracy and latency across risk categories[^3]. As of 2026 the repository is actively maintained, with roughly 7,100 stars and recent commits.

## Getting Started

```bash
pip install guardrails-ai
guardrails configure          # authenticate to the Hub (API token)
guardrails hub install hub://guardrails/regex_match
```

```python
from guardrails import Guard, OnFailAction
from guardrails.hub import RegexMatch

guard = Guard().use(
    RegexMatch,
    regex=r"\(?\d{3}\)?-? *\d{3}-? *-?\d{4}",
    on_fail=OnFailAction.EXCEPTION,
)

guard.validate("123-456-7890")        # passes
guard.validate("not a phone number")  # raises ValidationError
```

Structured output via Pydantic, with the model call wrapped by the Guard:

```python
from pydantic import BaseModel, Field
from guardrails import Guard
import openai

class Pet(BaseModel):
    pet_type: str = Field(description="Species of pet")
    name: str = Field(description="A unique pet name")

guard = Guard.for_pydantic(output_class=Pet)
res = guard(
    openai.chat.completions.create,
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Suggest a pet and a name."}],
)
print(res.validated_output)   # -> {"pet_type": "...", "name": "..."}
```

## Architecture / How It Works

A `Guard` is a pipeline with three stages: **pre-call** (input guards), the **LLM call** (which the Guard wraps and to which it can inject schema/format instructions), and **post-call** (output guards run over the raw completion). Each validator returns pass/fail plus, for failures, the offending spans and a fix value when one exists.

The **`on_fail` action** is the load-bearing design decision, because it determines control flow, not just severity:

- `EXCEPTION` — raise; the caller handles it.
- `FILTER` / `REFRAIN` — strip the failing content or blank the whole output.
- `FIX` — substitute the validator's programmatic fix (e.g. redact PII, truncate).
- `REASK` — send the failure back to the LLM with a correction prompt and re-run. This costs *additional* model calls, and the count is bounded by `num_reasks`. Re-asking is what makes structured-output enforcement work, and it is also the main hidden cost.
- `NOOP` — record the failure in the outcome object but pass the value through.

**Guardrails Hub** is a package registry, not a monolith. `guardrails hub install hub://guardrails/toxic_language` downloads that validator's code and, crucially, any ML model it depends on. Some validators are cheap (regex, competitor-name string match); others load transformer models (toxicity, PII/NER, semantic similarity) and pull in heavy dependencies. The Hub decoupling means your install footprint scales with how many model-backed validators you use, and validators can be versioned and updated independently of the core package.

For structured generation, the Guard translates a Pydantic model (or RAIL schema) into either a function-calling schema (for models that support it) or appended prompt instructions (for those that don't), then validates the parsed result and re-asks on schema violations[^1]. Streaming is supported: validators run incrementally over chunks, which for span-based validators means partial results can be filtered before the full completion arrives.

The **Guardrails Server** (`guardrails start`) exposes Guards over a Flask REST API, including an OpenAI-compatible endpoint so existing OpenAI SDK code can point `base_url` at a local Guard and get validation transparently[^1]. This is the recommended shape for multi-language or multi-service deployments.

## Production Notes

**Latency and cost are dominated by two things you control indirectly.** Model-backed validators add inference latency on every call — a toxicity or NER model runs locally (CPU by default) and can be the tail-latency driver, not the LLM. And `REASK` multiplies LLM calls: a Guard that re-asks up to 3 times can triple token spend and wall-clock time on the unlucky path. Budget for `num_reasks` explicitly and measure the reask rate in production, not just in tests.

**The Hub is a network and trust dependency.** `guardrails configure` requires a token, and `hub install` fetches code and model weights at install time. Air-gapped or reproducible builds need to vendor validators and pin versions; a fresh `hub install` is not hermetic. Validator quality varies — the Hub mixes first-party and community validators, and ML-based ones inherit their model's false-positive/false-negative profile. The Guardrails Index exists precisely because accuracy across validators is uneven[^3]; do not assume a validator's name implies production-grade precision without checking.

**API churn.** The 0.x line has had non-trivial breaking changes; the shift away from RAIL toward `Guard().use()` and `Guard.for_pydantic()` means a lot of older blog posts, Stack Overflow answers, and LLM-generated code target APIs that no longer match current signatures. Pin the version and read the version's own docs. Because the library is often used *by* LLM-assisted code, this is a frequent source of subtly wrong generated snippets.

**Validation is heuristic, not a guarantee.** Guardrails reduces the rate of bad outputs; it does not prove their absence. A jailbreak-detection or toxicity validator is a classifier with an error rate, and `FIX`/`FILTER` can mangle otherwise-good output. Treat it as defense-in-depth, not a security boundary — pair it with server-side authorization and output handling that assumes validators can be wrong.

**Server deployment.** The built-in `guardrails start` server is a dev server; the docs recommend Docker + Gunicorn for production[^1]. Each worker loads its own copy of any model-backed validators, so memory scales with worker count times model size — size instances accordingly.

## When to Use / When Not

**Use when:**
- You need runtime input/output guards (PII, toxicity, competitor mentions, topic restriction) composed declaratively and enforced on live traffic.
- You want a catalog of pre-built validators rather than writing and maintaining detectors yourself.
- You need structured output *plus* semantic validation, with automatic re-asking on failure.
- You want to expose validation as a language-agnostic service via an OpenAI-compatible endpoint.

**Avoid when:**
- You only need guaranteed-valid structured output — constrained-decoding and typed-extraction libraries are lighter and give stronger guarantees than re-asking.
- You need hard security guarantees; validators are probabilistic and belong behind real authorization.
- You want conversational dialogue rails (allowed topics, scripted flows) rather than field-level validators — that is a different tool's shape.
- Reproducible/air-gapped builds are mandatory and the Hub's install-time fetching is a non-starter without vendoring work.

## Alternatives

- NVIDIA/NeMo-Guardrails — use instead when you want dialogue-level rails and scripted conversation flows (Colang) rather than field validators.
- jxnl/instructor — use instead when you only need typed Pydantic output from LLMs, with re-asking, and no risk validation.
- dottxt-ai/outlines — use instead when you want constrained decoding that makes invalid structured output impossible rather than validated-after-the-fact.
- protectai/llm-guard — use instead when you want a security-focused prompt/response scanner (prompt injection, secrets, toxicity) without the Guard/structured-output abstraction.
- confident-ai/deepeval — use instead when the goal is offline evaluation and regression testing of LLM output, not runtime enforcement.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2023-01 | Initial public release; RAIL XML spec for schema + validators[^2]. |
| 0.2 | 2023 | Expanded validator set, Pydantic support growing. |
| 0.4 | 2024 | Guardrails Hub introduced; validators installed individually via CLI. |
| 0.5 | 2024 | Guardrails Server, OpenAI-compatible endpoint, streaming validation, `Guard().use()`-centric API[^1]. |
| 0.6 | 2025 | Continued Hub/server iteration; RAIL further de-emphasized. |
| — | 2025-02 | Guardrails Index benchmark launched (24 validators, 6 risk categories)[^3]. |

## References

[^1]: Guardrails AI documentation. https://www.guardrailsai.com/docs
[^2]: Guardrails README and project history, guardrails-ai/guardrails. https://github.com/guardrails-ai/guardrails
[^3]: Guardrails Index — validator accuracy/latency benchmark. https://index.guardrailsai.com

## Tags

python, llm, guardrails, validation, structured-output, ai-safety, pydantic, content-moderation, pii-detection, llm-ops
