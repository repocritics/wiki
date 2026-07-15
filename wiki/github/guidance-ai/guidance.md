# guidance-ai/guidance

> A Python DSL for constraining LLM output — regex, context-free grammars, and JSON schema enforced at decode time.

[GitHub repo](https://github.com/guidance-ai/guidance) ·
[PyPI](https://pypi.org/project/guidance/) ·
[License: MIT](https://github.com/guidance-ai/guidance/blob/main/LICENSE)

## Overview

Guidance is a library for steering language-model generation by imposing hard structural constraints on the tokens a model is allowed to emit. Instead of asking a model nicely for JSON and hoping, you describe the exact shape of valid output — a regex, a `select()` from a fixed list, a full context-free grammar, or a JSON schema — and Guidance masks the model's logits at each step so that only tokens consistent with the constraint can be sampled. Invalid output becomes structurally impossible rather than statistically unlikely.

It began as a Microsoft Research project led by Scott Lundberg (author of SHAP) and was released publicly in 2023[^1]. The early version used a Handlebars-style template string with `{{gen}}` / `{{select}}` mustache tags. That interface was later abandoned for the current Pythonic API, where a `Model` object is immutable and generation is expressed by overloading `+=` inside `system()` / `user()` / `assistant()` context managers[^2]. The project moved out of the `microsoft/` namespace into its own `guidance-ai` org. It remains widely referenced (over 21k stars) but development cadence is uneven — the constraint engine is now maintained largely as the separate `llguidance` Rust library.

The defining tension: Guidance's guarantees only hold when it can see and mask token logits. That requires a **local** backend (Transformers, llama.cpp) or an API that exposes constrained-decoding hooks. Against a black-box chat API like OpenAI's, most of the hard-constraint machinery degrades to best-effort or is unavailable.

## Getting Started

```bash
pip install guidance
```

```python
from guidance import models, gen, select, system, user, assistant

lm = models.Transformers("microsoft/Phi-4-mini-instruct")

with system():
    lm += "You are a geography expert."
with user():
    lm += "Capital of Sweden? Answer A) Helsinki B) Oslo C) Stockholm"
with assistant():
    lm += select(["A", "B", "C"], name="answer")     # only these 3 tokens are legal

print(lm["answer"])   # -> "C"
```

Constrain a `gen()` call to a regex, so the model literally cannot produce a non-integer:

```python
with assistant():
    lm += gen("age", regex=r"\d+", max_tokens=3)
```

## Architecture / How It Works

The core abstraction is a **stateless grammar**. Guidance functions decorated with `@guidance(stateless=True)` return grammar fragments — `gen()`, `select()`, string literals, `one_or_more()` — that compose into a single context-free grammar. `select(options=[a, b])` is alternation; concatenation is `+`; recursion is ordinary Python recursion. A JSON schema or Pydantic model is compiled down to a CFG under the hood, which is why `gen_json()` is "just" another grammar rather than a special code path.

At generation time the grammar is turned into a token-level automaton. Before each decode step Guidance computes which next tokens can still lead to a valid string and sets the logits of every other token to `-inf`, so sampling stays inside the grammar. Two consequences fall out of this:

- **Token fast-forwarding (acceleration).** When the grammar has only one legal continuation — e.g. the closing `</h1>` after `<h1>...` — Guidance emits those tokens directly without a forward pass through the model. This is a real latency/compute win on structured output and is a headline selling point over "ask for JSON and retry."
- **Token healing.** Constraints operate on characters, but models emit tokens, and a prompt boundary can fall mid-token. Guidance backs up over the last prompt token and re-generates it under the constraint, avoiding tokenization-boundary artifacts.

The constraint core has been extracted into **`llguidance`**, a Rust library that builds the token mask; the Python package is increasingly a front end over it. Backends are pluggable via `guidance.models` (`Transformers`, `LlamaCpp`, `Mock`, and API models). The `Mock` model plus `grammar.match(s)` lets you unit-test grammars offline with no model at all.

## Production Notes

- **Constraints need logit access.** The full guarantee set (regex/CFG/JSON masking, fast-forwarding, token healing) requires a local model where Guidance controls sampling. Remote chat APIs expose no per-token logit masking, so against them Guidance cannot enforce arbitrary grammars — check what your backend actually supports before assuming the constraint is hard.
- **Tokenizer coupling.** The token-level automaton is specific to the model's tokenizer. Constraint correctness and healing behavior depend on matching Guidance's view of the tokenizer to the backend's; mismatches surface as subtle off-by-one-token bugs, not clean errors.
- **CFG masking has per-step cost.** Computing the legal-token set at every position is not free. For tight grammars over large vocabularies the mask computation can rival model inference; the `llguidance` rewrite exists precisely to push that cost down.
- **Primary surface is Jupyter.** The rich output — highlighted fast-forwarded tokens, the generation widget — assumes a notebook. In plain scripts you get functional behavior but lose the visualization that makes debugging grammars tractable.
- **API churn.** Code written against the original Handlebars template syntax does not run on current Guidance; the `+=` / context-manager model is a full rewrite. Pin your version and expect migration work across major bumps.
- **Uneven maintenance.** Star count overstates momentum. Issue backlog is nontrivial (300+ open) and pushes are intermittent; treat it as a specialized tool with an active but small maintainer set, not a broadly-staffed platform.

## When to Use / When Not

**Use when:**
- You run a local/open-weights model and need output that provably matches a regex, grammar, or JSON schema.
- You want to cut latency/tokens on structured generation via fast-forwarding rather than generate-and-retry.
- Your task is a state machine — multiple choice, extraction into a fixed schema, controlled multi-step tool use — expressible as a grammar.

**Avoid when:**
- Your only backend is a hosted chat API with no constrained-decoding support — the core value evaporates; use the provider's own structured-output / function-calling instead.
- You need production-grade constrained decoding fused into a high-throughput server — Outlines and XGrammar-in-vLLM integrate more directly into serving stacks.
- You just want typed JSON from GPT-class endpoints — Instructor or native JSON mode is lighter weight.

## Alternatives

- dottxt-ai/outlines — regex/JSON/CFG constrained decoding with tighter serving-framework (vLLM) integration; use it when you need constraints inside a production inference server.
- guidance-ai/llguidance — the Rust constraint engine underneath Guidance; use it directly when embedding low-level token masking into your own stack.
- jxnl/instructor — Pydantic-validated structured output over API models; use it when you're on OpenAI/Anthropic and want typed responses without local logit control.
- 1rgs/jsonformer — narrow JSON-schema-only decoding; use it when your only need is filling a fixed JSON shape.
- sgl-project/sglang — programming model plus efficient serving for structured/multi-call LLM programs; use it when throughput and orchestration matter together.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-05 | Public release from Microsoft Research; Handlebars-style `{{gen}}`/`{{select}}` templates, token healing[^1]. |
| — | 2023 | Went viral for constrained generation + token fast-forwarding. |
| 0.1.x | 2023–2024 | Rewrite to the immutable-`Model` / `+=` / stateless-grammar API[^2]. |
| — | 2024 | Repo moved from `microsoft/` to the `guidance-ai` org. |
| — | 2024–2025 | Constraint core extracted into the `llguidance` Rust library[^3]. |

## References

[^1]: guidance-ai/guidance repository and release history — created 2022-11, public launch 2023. https://github.com/guidance-ai/guidance
[^2]: guidance README, current Pythonic API (`system()`/`user()`/`assistant()`, `gen`, `select`, `@guidance`). https://github.com/guidance-ai/guidance/blob/main/README.md
[^3]: llguidance — Rust constrained-decoding engine used by Guidance. https://github.com/guidance-ai/llguidance

## Tags

python, llm, constrained-decoding, structured-generation, grammar, json-schema, regex, prompt-engineering, inference, guidance
