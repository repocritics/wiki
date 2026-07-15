# flairNLP/flair

> A PyTorch NLP framework for sequence labeling — off-the-shelf NER, POS, and classification, plus its own contextual character-level embeddings.

[GitHub repo](https://github.com/flairNLP/flair) ·
[Official website](https://flairnlp.github.io/flair/) ·
[License: MIT](https://github.com/flairNLP/flair/blob/master/LICENSE)

## Overview

Flair is an NLP library built directly on PyTorch, started at Zalando Research in 2018 and now developed at Humboldt University of Berlin (Alan Akbik and collaborators)[^1]. Its original claim to fame was "Flair embeddings" — contextual string embeddings from a character-level bidirectional language model, introduced in the COLING 2018 paper that gives the project its name[^2]. For several years those embeddings, stacked with classical word vectors, held or approached state-of-the-art on standard NER benchmarks (CoNLL-03 across English, German, Dutch, Spanish).

The library's real value today is less the embeddings and more the packaging: a small, consistent object model (`Sentence`, `Token`, `Span`, `Label`) and one-line loading of pre-trained taggers for many languages and tasks. It is the fastest way to get a competitive NER or sentiment model running without touching model code, and one of the gentlest paths to training your own sequence tagger on custom labels. Many of its models are mirrored on the Hugging Face hub.

The defining tension is research-tool-vs-product. Flair is still pre-1.0 (0.15.1 as of late 2025)[^3], the API changes between minor versions, and its own LSTM-based embeddings are slow at inference relative to modern transformers. Since ~2020 the framework has leaned toward transformer fine-tuning (the "FLERT" approach[^4]) and few/zero-shot classification (TARS), which is where accuracy now comes from — the character LM is increasingly a legacy differentiator rather than the main draw.

## Getting Started

```bash
pip install flair    # requires Python 3.9+
```

```python
from flair.data import Sentence
from flair.nn import Classifier

sentence = Sentence("I love Berlin .")

# downloads and caches a pretrained model on first use (~/.flair)
tagger = Classifier.load("ner")
tagger.predict(sentence)

print(sentence)                        # Sentence[4]: "I love Berlin ." → ["Berlin"/LOC]
for entity in sentence.get_spans("ner"):
    print(entity.text, entity.tag, round(entity.score, 3))
```

Training a custom sequence tagger follows the same object model: load a `ColumnCorpus`, build a label dictionary, stack embeddings, wrap them in a `SequenceTagger`, and hand it to `ModelTrainer`.

## Architecture / How It Works

The data model is deliberately small. A `Sentence` holds `Token`s; predictions attach as `Label`s to tokens or to `Span`s (multi-token entities), each keyed by a label type (`"ner"`, `"pos"`, `"sentiment"`). Everything downstream — embeddings, taggers, trainers — reads and writes this structure.

Embeddings are the compositional core:

- **`FlairEmbeddings`** — the contextual string embeddings. A character-level bidirectional LSTM language model produces a per-token vector from the surrounding character stream, so the same word gets different vectors in different contexts[^2]. This is what made Flair distinctive; it is also comparatively slow because it runs an LSTM over characters.
- **`WordEmbeddings`** — classical static vectors (GloVe, fastText).
- **`TransformerWordEmbeddings` / `TransformerDocumentEmbeddings`** — wrap any Hugging Face transformer, with optional fine-tuning.
- **`StackedEmbeddings`** — concatenate several of the above into one vector. The canonical Flair recipe was Flair-forward + Flair-backward + GloVe stacked together.

Models sit on top of embeddings. `SequenceTagger` is a BiLSTM-CRF (or a transformer + linear/CRF head) for token-level tasks; `TextClassifier` pools document embeddings for sentence-level tasks. **TARS** (Task-Aware Representation of Sentences) reframes classification as an entailment problem to enable few-shot and zero-shot label sets[^5]. **HunFlair2** targets biomedical NER and entity linking.

`ModelTrainer` owns the training loop: learning-rate annealing on plateau, checkpointing, and evaluation. Pretrained models are resolved by short name (`"ner"`, `"pos"`, `"sentiment"`) to downloads that are cached under `~/.flair` and, increasingly, fetched from the Hugging Face hub.

## Production Notes

**Inference speed.** Flair embeddings are the slow path — a character LSTM per token does not vectorize the way a transformer forward pass does, and CPU inference on the classic stacked models is noticeably slower than spaCy or a batched transformer. If latency matters, prefer the transformer-based models, batch aggressively with `mini_batch_size`, and use a GPU. Flair is not built as a serving system; there is no built-in batching server, so teams typically wrap `predict` in FastAPI/Flask and manage concurrency themselves.

**Model downloads and caching.** First use of any model silently downloads weights to `~/.flair` (or the HF cache). In containerized or air-gapped deployments this is a footgun — bake the models into the image or pre-warm the cache, and pin model revisions rather than relying on short aliases that can move.

**Memory.** Loading a model loads the full network into memory; stacking multiple embeddings multiplies the vector width and the RAM/VRAM footprint. Large transformer document embeddings on long inputs are the usual OOM source during training.

**Dependency pinning.** Flair pins fairly tight ranges on `torch`, `transformers`, and tokenizers. Installing it into an existing ML environment frequently forces downgrades or conflicts; a dedicated virtualenv is the path of least resistance.

**Pre-1.0 API churn.** Being 0.x, minor releases have moved APIs — embedding class names, the `SequenceTagger.load` vs unified `Classifier.load` entry points, and training arguments have all shifted over the 0.x line. Pin the Flair version in `requirements.txt` and expect to touch code when upgrading. Reproducibility helpers (`flair.set_seed`) exist but full determinism on GPU is not guaranteed.

## When to Use / When Not

**Use when:**
- You want a competitive NER, POS, or sentiment model running in a few lines, especially for non-English languages.
- You need to train a custom sequence tagger on your own column-formatted data with minimal boilerplate.
- You are experimenting and want to stack/compare embeddings, or need few/zero-shot classification via TARS.
- You are doing biomedical NER (HunFlair2).

**Avoid when:**
- You need high-throughput, low-latency serving at scale — export a transformer to ONNX or reach for spaCy.
- You want a full production pipeline (tokenization, rules, components, serialization) — spaCy is more of a product.
- You only need raw transformer inference — use huggingface/transformers directly and skip the wrapper.
- You want a stable, long-supported API — the pre-1.0 churn is real.

## Alternatives

- explosion/spaCy — industrial-strength pipeline, much faster inference and a real production/serialization story; use it when throughput and a batteries-included pipeline matter more than swappable embeddings.
- huggingface/transformers — the underlying model zoo; use it directly when you want full control over the transformer and don't need Flair's data model.
- stanfordnlp/stanza — neural annotation across a very wide set of languages with a linguistics focus; use for multilingual dependency/morphology work.
- urchade/GLiNER — modern lightweight zero-shot NER for arbitrary entity types; use when you need flexible, prompt-like entity schemas.
- nltk/nltk — classical, teaching-oriented toolkit; use for coursework and traditional algorithms rather than SOTA models.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2018 | Initial release at Zalando Research; contextual string embeddings[^2]. |
| — | 2019 | FLAIR framework paper (NAACL 2019 demonstrations)[^1]. |
| — | 2020 | FLERT (document-level transformer features)[^4] and TARS few/zero-shot[^5]. |
| 0.11 | 2022 | Continued transformer-first shift; biomedical HunFlair line. |
| 0.15.1 | 2025-10 | Current release; Python 3.9+[^3]. Last push 2025-10-27. |

As of mid-2026 the repo is actively maintained (last push October 2025, ~14.4k stars, ~2.1k forks) with a low open-issue count (~32), consistent with a mature, narrowly-scoped library rather than a fast-moving platform.

## References

[^1]: Akbik et al., "FLAIR: An Easy-to-Use Framework for State-of-the-Art NLP" — NAACL 2019 (Demonstrations). https://aclanthology.org/N19-4010/
[^2]: Akbik, Blythe, Vollgraf, "Contextual String Embeddings for Sequence Labeling" — COLING 2018. https://aclanthology.org/C18-1139/
[^3]: Flair releases. https://github.com/flairNLP/flair/releases
[^4]: Schweter and Akbik, "FLERT: Document-Level Features for Named Entity Recognition" — 2020. https://arxiv.org/abs/2011.06993
[^5]: Halder et al., "Task-Aware Representation of Sentences for Generic Text Classification" — COLING 2020. https://aclanthology.org/2020.coling-main.285/

## Tags

python, nlp, named-entity-recognition, sequence-labeling, pytorch, word-embeddings, text-classification, machine-learning, transformers, deep-learning
