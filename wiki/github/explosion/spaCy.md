# explosion/spaCy

> Industrial-strength NLP for Python — one opinionated pipeline per task, tuned for throughput on production text.

[GitHub repo](https://github.com/explosion/spaCy) ·
[Official website](https://spacy.io) ·
[License: MIT](https://github.com/explosion/spaCy/blob/master/LICENSE)

## Overview

spaCy is a natural language processing library written in Python and Cython, first released as 1.0 in 2016 by Matthew Honnibal and the company Explosion[^1]. Its design premise is the opposite of a research toolkit: where NLTK exposes many algorithms for teaching and experimentation, spaCy ships exactly one implementation per task — the one its authors judged best for production — and optimizes it heavily. The result is a library you assemble into an application, not a menu of interchangeable academic components.

The unit of work is the `nlp` pipeline: raw text goes in, an annotated `Doc` object comes out. A `Doc` carries tokenization, part-of-speech tags, dependency parse, named entities, lemmas, and any custom attributes, all sharing one underlying token array. Components (tagger, parser, `ner`, `lemmatizer`, `textcat`, and user-defined ones) run in sequence and mutate the `Doc` in place. spaCy supports tokenization and training for 70+ languages and distributes pretrained pipelines as ordinary pip-installable Python packages[^2].

The defining tension is opinionation versus flexibility. spaCy is fast and consistent because it refuses to be a general framework; you get its tokenizer, its parser, its config system. When your problem fits — entity extraction, linguistic features, document classification at scale — this is a strength. When you need cutting-edge research models or capabilities spaCy declines to include (coreference, sentiment, generative tasks), you reach outside it, and spaCy is happiest acting as the orchestration layer rather than the model.

## Getting Started

```bash
pip install -U pip setuptools wheel
pip install spacy
python -m spacy download en_core_web_sm
```

```python
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("Explosion, based in Berlin, ships spaCy and Prodigy.")

for ent in doc.ents:
    print(ent.text, ent.label_)      # Explosion ORG, Berlin GPE ...

for tok in doc:
    print(tok.text, tok.pos_, tok.dep_, tok.head.text)

# Batch processing — nlp.pipe streams and batches for throughput
texts = ["first document", "second document"]
for doc in nlp.pipe(texts, batch_size=64):
    print([e.text for e in doc.ents])
```

## Architecture / How It Works

**Cython core.** The hot path is not Python. A `Doc` wraps a C array of `TokenC` structs; `Token` and `Span` are lightweight views into that array rather than owned objects. Strings are interned in a `StringStore` and referenced by 64-bit hashes throughout, so the pipeline moves integers, not Python strings. This is why spaCy is fast and why building from source needs a C compiler (prebuilt wheels avoid this for common platforms).

**Language and pipeline.** `spacy.load()` returns a `Language` object holding a shared `Vocab` and an ordered list of pipeline components. Each component implements `__call__(doc) -> doc`. Components are registered by name through `catalogue`/`confection` registries, which is how the config system instantiates them from strings.

**Thinc, not PyTorch directly.** The statistical models are built on Thinc[^3], Explosion's own lightweight ML library with a functional layer API. Thinc can wrap PyTorch or TensorFlow layers, so transformer components ultimately call Hugging Face `transformers` under the hood, but the spaCy-facing API is Thinc's.

**Config-driven training (v3).** Training is described by a single `config.cfg` file that fully specifies the pipeline, model architectures, optimizer, and data paths[^4]. `spacy train` reads the config, there are no hidden defaults in code. This makes runs reproducible but adds a learning curve: the config format and registry system are their own thing to learn.

**Transformer listener pattern.** With `trf` pipelines, one `Transformer` component runs the model once per doc and downstream components (tagger, parser, ner) "listen" to its output via a shared reference rather than each re-embedding the text. This is what keeps multi-task transformer pipelines affordable, and it is also a coupling that complicates swapping components in and out.

## Production Notes

**Models are pinned to spaCy's minor version.** A model trained or published for spaCy 3.7 is not guaranteed to load on 3.8. After upgrading, run `python -m spacy validate` to check installed pipelines, and re-download or retrain. Models you trained yourself should be retrained after a spaCy upgrade — the runtime and training versions must match. This version coupling is the single most common upgrade surprise.

**The v2 to v3 jump was a genuine rewrite.** spaCy 3.0 (2021) replaced the training API with the config system, changed the serialized model format, and altered `nlp.add_pipe` signatures. Code and models did not carry over transparently; there is a dedicated migration guide[^5]. Treat a 2.x to 3.x move as a port, not an upgrade.

**Throughput and parallelism.** Use `nlp.pipe()` for anything beyond a handful of texts — it batches and avoids per-call overhead. `n_process` forks worker processes, which multiplies memory (each worker holds its own copy of the pipeline and vectors) and is fork-based, so it interacts poorly with some environments and with already-multithreaded BLAS. Prefer batching before reaching for multiprocessing.

**Transformer pipelines need a GPU to be practical.** The `sm`/`md`/`lg` statistical models are CPU-friendly and fast; `trf` models are accurate but slow on CPU and expect `spacy[cuda]` plus a compatible CuPy/CUDA stack. The `lg` models are large mainly because of bundled word vectors — that is memory, not accuracy, and `md` is often the better default.

**Scope limits to plan around.** Core spaCy has no coreference resolution, no sentiment analysis, and no text generation. Coreference lived in `spacy-experimental` and was never folded into a stable core component. For anything generative or LLM-shaped, `spacy-llm` wraps hosted and local models as pipeline components, but spaCy is the glue there, not the model.

**Python version ceiling.** spaCy tracks a supported Python range and historically lags the newest interpreter by a release while wheels and dependencies catch up (the 3.8 line documented Python >=3.7,<3.13). Check the support matrix before pinning a bleeding-edge Python.

## When to Use / When Not

**Use when:**
- You need named-entity recognition, POS/dependency features, or document classification on real volumes of text.
- You want one fast, consistent pipeline you can package and deploy, not a research sandbox.
- You need custom pipeline components and per-token custom attributes with a stable API.
- You want reproducible, config-defined training over your own labeled data.

**Avoid when:**
- Your task is generative, conversational, or best served by calling an LLM directly.
- You need capabilities spaCy deliberately omits (coreference, sentiment) and don't want to bolt on extensions.
- You want to compare many algorithms per task — spaCy gives you one on purpose.
- You're doing linguistics teaching or exploratory research where NLTK's breadth fits better.

## Alternatives

- nltk/nltk — broad, teaching-oriented toolkit; use when you want many algorithms and classroom breadth over production speed.
- huggingface/transformers — use when the task is model-centric (fine-tuning, generation, SOTA benchmarks) rather than pipeline orchestration.
- stanfordnlp/stanza — use when you need research-grade accuracy across many languages and can accept slower throughput.
- flairNLP/flair — use when contextual-embedding sequence tagging is the whole job and you want a simpler API.
- explosion/spacy-llm — use when you want spaCy's pipeline structure but LLM-backed components instead of trained statistical models.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-10 | First stable release; Cython core, tokenizer, parser, NER[^1]. |
| 2.0 | 2017-11 | Neural network models via Thinc, multi-language support expansion. |
| 2.1 | 2019-02 | Improved models, pretraining (`spacy pretrain`). |
| 3.0 | 2021-02 | Config system, transformer pipelines, `spacy projects`, Typer CLI[^4]. |
| 3.7 | 2023-10 | Continued 3.x line; dependency and model refreshes. |
| 3.8 | 2024 | Current release line at time of writing[^2]. |

## References

[^1]: Explosion, "spaCy — Industrial-Strength Natural Language Processing." https://spacy.io
[^2]: spaCy README and release notes, explosion/spaCy (v3.8 line). https://github.com/explosion/spaCy/releases
[^3]: Thinc — the machine learning library behind spaCy. https://thinc.ai
[^4]: spaCy, "What's New in v3.0." https://spacy.io/usage/v3
[^5]: spaCy, "Migrating from v2.x to v3.x." https://spacy.io/usage/v3#migrating

## Tags

python, cython, nlp, natural-language-processing, named-entity-recognition, text-classification, tokenization, machine-learning, transformers, dependency-parsing, production-nlp
