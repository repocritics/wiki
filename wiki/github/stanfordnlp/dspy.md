# stanfordnlp/dspy

> Programming language models with typed modules and automatic prompt optimization instead of hand-tuned prompt strings.

[GitHub repo](https://github.com/stanfordnlp/dspy) ·
[Official website](https://dspy.ai) ·
[License: MIT](https://github.com/stanfordnlp/dspy/blob/main/LICENSE)

## Overview

DSPy ("Declarative Self-improving Python") is a Python framework for building LM systems as composable, typed modules rather than as strings of prompt text[^1]. You declare what a step should do — its input and output fields — as a *signature*, wire signatures into *modules*, and then hand the whole program to an *optimizer* that searches for the few-shot demonstrations and instruction wording that maximize a metric you define. The stated thesis is that prompt engineering is a compilation target, not an authoring task: you write the program, DSPy compiles the prompts.

The project began as **Demonstrate-Search-Predict (DSP)** at Stanford NLP, led by Omar Khattab, and was renamed DSPy in August 2023[^2]. Its ideas were formalized in the ICLR 2024 paper "DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines"[^3]. With ~36k stars and ~3.1k forks it is one of the most-used frameworks in the "structured LM programming" space, and it is developed actively — the repository sees near-daily pushes and carries a large open-issue backlog (~590), typical of a fast-moving project even after a 3.0 release.

The defining tradeoff is indirection. DSPy trades the immediacy of a raw prompt (you can read exactly what the model sees) for a layer of signatures, adapters, and optimizers that generate that prompt for you. When optimization works it removes tedious hand-tuning; when it does not, you are debugging generated prompts through several abstraction layers.

## Getting Started

```bash
pip install dspy
```

```python
import dspy

# LM access goes through LiteLLM; any supported provider string works.
dspy.configure(lm=dspy.LM("openai/gpt-4o-mini"))

# A signature can be a string: "inputs -> outputs".
qa = dspy.ChainOfThought("question -> answer")
print(qa(question="What is the capital of France?").answer)
```

Optimizing a program against a metric:

```python
from dspy import Example
from dspy.teleprompt import BootstrapFewShot

trainset = [Example(question=q, answer=a).with_inputs("question")
            for q, a in my_pairs]

def exact_match(example, pred, trace=None):
    return example.answer.lower() == pred.answer.lower()

compiled = BootstrapFewShot(metric=exact_match).compile(qa, trainset=trainset)
```

## Architecture / How It Works

DSPy is built from four layers that are deliberately separable:

1. **Signatures** — declarative specs of a step's inputs and outputs. A signature can be a terse string (`"question -> answer"`) or a `dspy.Signature` subclass with typed `InputField`/`OutputField` and docstring instructions. Signatures describe intent; they do not contain prompt text.
2. **Modules** — the callable units. `dspy.Predict` is the base module; `ChainOfThought` adds a reasoning field, `ReAct` runs a tool-use loop, `ProgramOfThought` generates and executes code. Modules compose like PyTorch `nn.Module`s — you subclass `dspy.Module`, instantiate sub-modules in `__init__`, and call them in `forward`.
3. **Adapters** — translate a signature + inputs into the actual provider request and parse the response back into typed fields. `ChatAdapter` is the default; `JSONAdapter` uses structured-output/JSON mode where the provider supports it. This layer is why the same program runs across providers.
4. **Optimizers (teleprompters)** — take a program, a metric, and a trainset, and search for better prompts or weights. `BootstrapFewShot` self-generates few-shot demonstrations by running the program and keeping traces that pass the metric. `MIPROv2` jointly searches instructions and demonstrations with a Bayesian-style proposal loop[^4]. `BootstrapFinetune` compiles a prompt program into fine-tuning data. `GEPA` uses reflective natural-language evolution of prompts and reports beating RL baselines on some tasks[^5].

Provider access is delegated to **LiteLLM**, so DSPy itself carries no per-provider client code — a model is just a string like `openai/gpt-4o` or `anthropic/claude-...`. Caching, retries, and token accounting ride on that layer.

The mental model is compiler-like: a program is the source, the optimizer is the compiler pass, and the artifact is a set of prompts (and optionally finetuned weights) you can save and reload. This is the source of both DSPy's leverage and its opacity.

## Production Notes

**Optimization is expensive and stochastic.** Running an optimizer issues many LM calls (bootstrapping traces, evaluating candidates across the trainset). Cost and wall-clock time scale with trainset size and candidate count, and because the search is metric-driven over a noisy model, two compile runs can yield different prompts. Budget for it and cache aggressively.

**Save the compiled artifact, don't recompile in prod.** Use `program.save()` / `load()` to persist the optimized state (demonstrations and instructions) and ship that. Recompiling on deploy reintroduces cost and nondeterminism.

**The generated prompt is the real prompt.** When output is wrong, inspect what was actually sent. `dspy.inspect_history(n=1)` prints the last rendered prompt(s); this is the fastest way to see through the adapter layer. Parsing failures usually trace to a signature whose field types the adapter could not reliably extract.

**Metric design is the hard part.** DSPy optimizes exactly what you measure. A weak or gameable metric produces a program that scores well and behaves badly. LM-as-judge metrics add their own cost and variance to every optimization step.

**API churn.** DSPy has moved fast and broken interfaces between minor versions: the LM client was reworked around LiteLLM, adapters were introduced, and the `dspy.Assert`/`Suggest` assertion API was reworked in the 2.6→3.0 line. Pin a version and read the changelog before upgrading; tutorials written for older releases frequently no longer run verbatim.

**Structured output reliability varies by provider.** `JSONAdapter` leans on the provider's JSON/structured-output support; on models without it, fall back to `ChatAdapter` and expect more parsing edge cases with complex typed fields.

## When to Use / When Not

**Use when:**
- You have a measurable metric and a modest labeled set, and want the system to tune prompts/demonstrations instead of doing it by hand.
- You are building multi-stage pipelines (RAG, agent loops, classifiers) and want typed composition rather than string concatenation.
- You want provider portability and the ability to later fine-tune from the same program.

**Avoid when:**
- You have a single, stable prompt that already works — the abstraction overhead buys you nothing.
- You need to read and control the exact prompt bytes at all times (regulated or latency-critical prompt paths).
- You cannot define a meaningful automatic metric; without one, the optimizers — DSPy's main value — have nothing to optimize against.

## Alternatives

- microsoft/guidance — constrained generation and templating with tight control over the exact token stream; use when you want to script the prompt precisely, not have it compiled for you.
- guardrails-ai/guardrails — validation and structured-output enforcement; use when your problem is output correctness/schema, not prompt optimization.
- BerriAI/litellm — the provider-abstraction layer DSPy sits on; use directly when you only need unified multi-provider calls without the programming model.
- langchain-ai/langchain — broad orchestration and integrations; use when you want a large connector ecosystem and are comfortable hand-writing prompts.
- jxnl/instructor — Pydantic-typed structured extraction; use when you want typed outputs from an LM without DSPy's optimizer machinery.

## History

| Version | Date | Notes |
|---------|------|-------|
| DSP | 2022-12 | Demonstrate-Search-Predict released; the precursor library[^2]. |
| DSPy (rename) | 2023-08 | Framework renamed and repositioned around signatures/modules[^2]. |
| — | 2024-05 | ICLR 2024 paper published; MIPRO/MIPROv2 optimizer line[^3][^4]. |
| 2.5 | 2024-09 | LM client reworked around LiteLLM; adapters introduced. |
| 2.6 | 2025 | Continued adapter/optimizer refinement. |
| 3.0 | 2025 | Major line; assertion API reworked, GEPA optimizer added[^5]. |

## References

[^1]: DSPy documentation. https://dspy.ai
[^2]: DSPy repository README and project history, Stanford NLP. https://github.com/stanfordnlp/dspy
[^3]: Khattab et al., "DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines," ICLR 2024. https://arxiv.org/abs/2310.03714
[^4]: Opsahl-Ong et al., "Optimizing Instructions and Demonstrations for Multi-Stage Language Model Programs," 2024. https://arxiv.org/abs/2406.11695
[^5]: Agrawal et al., "GEPA: Reflective Prompt Evolution Can Outperform Reinforcement Learning," 2025. https://arxiv.org/abs/2507.19457

## Tags

python, llm, prompt-optimization, prompt-engineering, agents, rag, framework, machine-learning, stanford-nlp, declarative
