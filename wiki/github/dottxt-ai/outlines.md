# dottxt-ai/outlines

> Constrained decoding for LLMs — force a model's output to match a regex, JSON Schema, or grammar during generation instead of parsing and retrying after.

[GitHub repo](https://github.com/dottxt-ai/outlines) ·
[Official website](https://dottxt-ai.github.io/outlines/) ·
[License: Apache-2.0](https://github.com/dottxt-ai/outlines/blob/main/LICENSE)

## Overview

Outlines is a Python library for *structured generation*: rather than prompting a
model to "return JSON" and hoping, it intercepts the token sampling loop and masks
out every token that would violate the target structure, so the only sequences the
model can produce are valid ones. It was built by the team at .txt (dottxt) and grew
directly out of the paper "Efficient Guided Generation for Large Language Models"[^1],
whose contribution was showing that a regex or grammar can be compiled to a finite
state machine and the vocabulary indexed against it so the per-token masking cost is
roughly constant rather than proportional to vocabulary size.

The defining tension is that Outlines only works where it can see and edit logits.
For local backends (transformers, llama.cpp, vLLM, Ollama servers) it constrains
generation token-by-token and the output is guaranteed to parse. For closed API
providers (OpenAI, Gemini) it can only lean on whatever native structured-output mode
the provider exposes — the guarantee is weaker and the mechanism is different. The
same `model(prompt, output_type)` call spans both worlds, which is convenient but
hides a real difference in how much enforcement is actually happening underneath.

The library went through a substantial API rewrite. Older code used
`outlines.models.transformers(...)` plus a `outlines.generate.json/regex/cfg` family;
the current interface is `outlines.from_transformers(...)` (or `from_vllm`,
`from_llamacpp`, `from_openai`, etc.) returning a callable you invoke with an output
type[^2]. Most tutorials and Stack Overflow answers online still show the old API.

## Getting Started

```shell
pip install outlines
```

```python
import outlines
from transformers import AutoTokenizer, AutoModelForCausalLM

MODEL = "microsoft/Phi-3-mini-4k-instruct"
model = outlines.from_transformers(
    AutoModelForCausalLM.from_pretrained(MODEL, device_map="auto"),
    AutoTokenizer.from_pretrained(MODEL),
)

from typing import Literal
from pydantic import BaseModel

class Review(BaseModel):
    rating: Literal["poor", "fair", "good", "excellent"]
    pros: list[str]
    cons: list[str]

# output_type drives constrained decoding; result parses by construction
out = model(
    "Review: great battery, runs hot, poor webcam.",
    Review,
    max_new_tokens=200,
)
review = Review.model_validate_json(out)
```

The output type can be a Python type (`int`, `Literal[...]`), a Pydantic model, a raw
regex, a JSON Schema, a context-free grammar, or a typed function signature (from
which the parameter schema is inferred).

## Architecture / How It Works

The pipeline is: your target (Pydantic model / JSON Schema / regex / grammar) is
lowered to a regular expression or a context-free grammar, that is compiled to a
finite state machine, and the FSM is *indexed against the model's tokenizer* — for
each state, which tokens keep the string valid. At every decode step Outlines looks
up the current FSM state, builds a boolean mask over the vocabulary, and sets the
logits of disallowed tokens to `-inf` before sampling. Because the index is
precomputed, the runtime cost per token is small; the cost moves to the one-time
compile of the index[^1].

JSON is handled by translating JSON Schema to a regex that encodes field order,
types, and enum values, then the regex path applies. Grammars use a parser (Lark
syntax) with incremental/partial parsing so the FSM can advance as tokens stream in;
this path is more general but historically less battle-tested than regex and JSON.

The performance-critical core — FSM construction and vocabulary indexing — was
extracted into a separate Rust crate, `outlines-core`, with Python bindings, so the
indexing engine can be reused by other projects independent of the Python
front-end[^3]. The index is specific to a `(tokenizer, schema)` pair and is cached;
switching models with different tokenizers invalidates it.

Because enforcement lives entirely in logit masking, Outlines is inference-engine
agnostic but inference-engine *dependent*: it needs a backend that hands it logits.
This is why it integrates cleanly with vLLM and transformers and only loosely with
hosted APIs.

## Production Notes

- **Index compilation is the latency you don't see in a hello-world.** For large or
  deeply nested JSON Schemas and for grammars, building the FSM index can take from
  hundreds of milliseconds to several seconds and consume noticeable memory. It is a
  one-time, cacheable cost per `(tokenizer, schema)`, so warm it at startup rather
  than on the first user request.
- **Constraining can lower answer quality, not just format validity.** Forcing tokens
  into a schema the model finds unnatural (odd key order, tight enums, coercing a
  reasoning task into a rigid shape) can push it off its preferred distribution.
  Structured output guarantees the shape, not the correctness of what's inside.
- **Whitespace and unbounded fields are classic footguns.** Permissive JSON regexes
  historically let the model emit long runs of whitespace or pad string fields;
  bound your string lengths and array sizes in the schema instead of trusting
  defaults.
- **The API rewrite is a real migration.** Code and blog posts using
  `outlines.generate.json(...)` / `outlines.models.transformers(...)` will not run
  against the current `from_*` + `model(prompt, output_type)` interface. Pin a
  version and read the changelog before upgrading across the boundary[^2].
- **In vLLM you have competing backends.** vLLM has supported multiple
  structured-output engines (Outlines, lm-format-enforcer, and the grammar-based
  xgrammar); which one is active and fastest depends on your vLLM version and config,
  so benchmark on your own schemas rather than assuming Outlines is the default[^4].
- **Grammar (CFG) mode is the least mature path** — expect more edge cases than the
  regex/JSON paths, especially with complex recursive grammars.

## When to Use / When Not

**Use when:**
- You run models locally or through vLLM/Ollama and need outputs that are guaranteed
  to parse (JSON, a regex-shaped ID, a fixed enum) without a retry loop.
- You want one calling convention across transformers, llama.cpp, vLLM, and some APIs.
- You need regex- or grammar-level control, not just "valid JSON."

**Avoid when:**
- You only ever call a hosted API (OpenAI/Gemini) — their native structured-output or
  a validation-and-retry library like instructor may be simpler, since Outlines can't
  mask logits you don't control.
- Your task is open-ended reasoning where hard constraints would fight the model.
- You need a mature grammar engine as the central feature — evaluate xgrammar too.

## Alternatives

- guidance-ai/guidance — constrained generation plus templating/control flow; broader
  programming model, use when you want interleaved generation and logic.
- noamgat/lm-format-enforcer — token-filtering enforcement, also integrated in vLLM;
  use when you want an alternative masking backend in the same stack.
- mlc-ai/xgrammar — fast grammar-based structured generation; use when grammar/CFG
  performance is the priority (also a vLLM/SGLang backend).
- jxnl/instructor — validation-and-retry over API providers via Pydantic; use when you
  live entirely on hosted APIs and can't touch logits.
- 1rgs/jsonformer — minimal JSON-only field-by-field filling; use for simple JSON
  extraction without the rest of the machinery.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-03 | First public release by .txt; regex/JSON/CFG guided generation[^1]. |
| paper | 2023-07 | "Efficient Guided Generation for LLMs" — FSM vocabulary indexing[^1]. |
| outlines-core | 2024 | FSM/indexing core split into a Rust crate with Python bindings[^3]. |
| 0.1.x | 2024 | Restructured internals around outlines-core. |
| v1 API | 2025 | Unified `from_*` loaders + `model(prompt, output_type)` calling convention[^2]. |

## References

[^1]: Brandon T. Willard, Rémi Louf, "Efficient Guided Generation for Large Language Models" — arXiv:2307.09702. https://arxiv.org/abs/2307.09702
[^2]: Outlines documentation (current API). https://dottxt-ai.github.io/outlines/
[^3]: outlines-core (Rust core for FSM construction and vocabulary indexing). https://github.com/dottxt-ai/outlines-core
[^4]: vLLM documentation, structured / guided decoding backends. https://docs.vllm.ai/

## Tags

python, structured-generation, llm, constrained-decoding, json-schema, regex, finite-state-machine, pydantic, guided-generation, inference, dottxt
