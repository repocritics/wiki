# piskvorky/gensim

> Memory-efficient, streaming topic modelling and word-embedding library for Python — now in stated maintenance mode.

[GitHub repo](https://github.com/piskvorky/gensim) ·
[Official website](https://radimrehurek.com/gensim) ·
[License: LGPL-2.1](https://github.com/piskvorky/gensim/blob/develop/COPYING)

## Overview

Gensim ("Generate Similar") is a Python library for unsupervised topic modelling, document indexing, and similarity retrieval over large text corpora. It grew out of Radim Řehůřek's academic work and was introduced in a 2010 LREC workshop paper[^1]; the codebase predates the modern NLP stack (the repo dates to 2011) and long served as the default tool for Latent Semantic Indexing (LSI), Latent Dirichlet Allocation (LDA), and — after 2013 — the reference Python `word2vec`/`doc2vec`/`fastText` implementations.

The library's defining design goal is **memory independence**: every algorithm streams its corpus one document at a time and never requires the whole dataset to fit in RAM[^2]. A corpus in gensim is any Python iterable that yields sparse bag-of-words vectors, so training LSI on a 100 GB Wikipedia dump on a laptop is a first-class use case rather than an afterthought. This streaming/iterator ethos is what distinguished gensim from scikit-learn's in-memory matrices, and it remains its clearest reason to exist.

As of 2026 gensim is explicitly in **stable maintenance mode**: the maintainers accept bug fixes and documentation but no new features[^3]. The field moved to transformer-based dense embeddings (BERT, sentence-transformers), and gensim's word2vec/LSI/LDA algorithms are now considered classical baselines rather than state of the art. It is still widely depended upon — as a fast, dependency-light way to train or load word vectors and run classical topic models — but it is not where new NLP research lands.

## Getting Started

```bash
pip install --upgrade gensim   # pulls NumPy, SciPy, smart_open
```

```python
from gensim import corpora, models, similarities

docs = [
    "human interface computer",
    "survey user computer system response time",
    "graph minors survey",
]
texts = [d.split() for d in docs]

dictionary = corpora.Dictionary(texts)          # token -> integer id
bow = [dictionary.doc2bow(t) for t in texts]     # streamed sparse vectors

tfidf = models.TfidfModel(bow)                   # fit weighting
lsi = models.LsiModel(tfidf[bow], id2word=dictionary, num_topics=2)

index = similarities.MatrixSimilarity(lsi[tfidf[bow]])
query = dictionary.doc2bow("computer system".split())
print(list(enumerate(index[lsi[tfidf[query]]])))  # cosine sims vs corpus
```

```python
# word2vec: train embeddings from a streamed iterable of token lists
from gensim.models import Word2Vec

model = Word2Vec(sentences=texts, vector_size=100, window=5,
                 min_count=1, workers=4)
model.wv.most_similar("computer")     # KeyedVectors API (gensim 4.x)
```

## Architecture / How It Works

Gensim is organized around three composable abstractions:

1. **Dictionary** — maps tokens to integer ids and tracks document frequencies. Can be built incrementally and pruned (`filter_extremes`).
2. **Corpus** — any iterable of sparse `(id, count)` vectors. Backed on disk by Matrix Market (`.mm`), or streamed from a generator. Nothing is loaded eagerly.
3. **Model / transformation** — objects like `TfidfModel`, `LsiModel`, `LdaModel`, `HdpModel` implement a uniform `model[corpus]` transform interface, so transformations chain lazily (`lsi[tfidf[bow]]`).

Under the hood, "pure Python" is misleading. The linear-algebra-heavy models (LSI, LDA) delegate to NumPy/SciPy, which in turn call into a BLAS/LAPACK backend (OpenBLAS, MKL, Accelerate)[^2]. The neural models — `word2vec`, `doc2vec`, `fastText` — ship hand-written **Cython** inner training loops (`word2vec_inner.pyx` and friends) that release the GIL and parallelize across `workers` threads. When the compiled Cython extension is missing, gensim silently falls back to a pure-Python path that is roughly two orders of magnitude slower.

Word vectors live in a **`KeyedVectors`** object (`model.wv`), a thin wrapper over a dense NumPy array plus a vocab index. Since gensim 4.0 this is the single canonical embedding container, used for word2vec, fastText, and vectors imported from external formats (GloVe, Facebook `.bin`). The `gensim.downloader` API fetches pretrained models and corpora from a hosted store. `smart_open` is a hard dependency because every load/save path can transparently be a local file, S3 URL, HTTP URL, or gzip stream.

## Production Notes

**The C-extension warning is real.** If you see `"C extension not loaded, training will be slow"`, the Cython accelerators did not compile — usually a missing C compiler or a broken wheel/source mismatch. Training word2vec in that state can be ~100× slower. Verify with `from gensim.models.word2vec import FAST_VERSION; print(FAST_VERSION)` (should be `0` or `1`, not `-1`).

**SciPy coupling is a recurring break.** Gensim reaches into SciPy internals, and SciPy's removals have broken gensim repeatedly — most visibly when SciPy 1.13 removed `scipy.linalg.triu`, causing `ImportError` on older gensim. Pin compatible versions or upgrade to a gensim release that fixed the specific breakage; do not assume an arbitrary NumPy/SciPy pairing works.

**The 4.0 migration was not backward-compatible.** gensim 4.0.0 (2021) was a large cleanup: Python 2 support removed, the `word2vec`/`doc2vec` APIs reorganized around `KeyedVectors`, many deprecated call sites removed, and the built-in text-summarization module dropped entirely[^4]. Models pickled with gensim 3.x frequently fail to `load()` under 4.x. Treat a 3→4 upgrade as a code-and-artifact migration, not a version bump.

**Memory independence applies to the corpus, not the model.** Streaming keeps input off-heap, but a trained model's parameters (LDA topic-word matrix, the full word-vector array) sit in RAM. Large vocabularies with high `vector_size` can still exhaust memory; prune the dictionary and cap vocab with `max_final_vocab`/`min_count`.

**BLAS matters.** LSI/LSA throughput is dominated by the underlying BLAS. A generic reference BLAS versus OpenBLAS/MKL can differ by an order of magnitude on the same hardware.

**Reproducibility.** word2vec results are only deterministic with `workers=1` and a fixed `seed`; multithreaded training is inherently nondeterministic due to unsynchronized updates.

## When to Use / When Not

**Use when:**
- You need to train or serve classical embeddings (word2vec/fastText/doc2vec) with a small dependency footprint and no GPU.
- Your corpus is larger than RAM and you want streaming, out-of-core LSI/LDA/TF-IDF.
- You want fast, well-worn implementations of LDA/LSI for interpretable topic models.
- You need to load pretrained vectors (GloVe, fastText `.bin`) into a uniform `KeyedVectors` API.

**Avoid when:**
- You want state-of-the-art semantic search or classification — dense transformer embeddings (sentence-transformers) beat word2vec/LSI substantially.
- You need an actively evolving library; gensim is in maintenance mode and will not add features.
- Your topic-modelling pipeline already lives in scikit-learn — its LDA/NMF may be simpler to integrate than adding a second corpus abstraction.
- You need GPU-accelerated training; gensim is CPU/Cython only.

## Alternatives

- huggingface/transformers — modern contextual embeddings; use when you need transformer-quality semantics rather than static word vectors.
- UKPLab/sentence-transformers — sentence/document embeddings for similarity and retrieval; use when replacing LSI/word2vec-averaged search.
- scikit-learn/scikit-learn — LDA, NMF, TF-IDF, TruncatedSVD in one toolkit; use when topic modelling is one step in a broader ML pipeline.
- MaartenGr/BERTopic — topic modelling over transformer embeddings + clustering; use when you want interpretable topics with modern embeddings.
- facebookresearch/fastText — the original C++ fastText; use when you want the reference implementation and CLI rather than the Python port.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.2.0 | 2010 | Early public releases; LSI/LDA over streamed corpora[^1]. |
| 1.0.0 | 2017-02 | API stabilization; deprecation cleanups. |
| 3.0.0 | 2017-09 | Major line; expanded word2vec/fastText/keyedvectors work. |
| 3.8.3 | 2020 | Last release supporting Python 2.7. |
| 4.0.0 | 2021-03 | Python-3-only rewrite; `KeyedVectors` unification; summarization removed[^4]. |
| 4.3.x | 2022–2024 | Newer Python and SciPy/NumPy compatibility fixes; maintenance line[^3]. |

## References

[^1]: Radim Řehůřek and Petr Sojka, "Software Framework for Topic Modelling with Large Corpora" — LREC 2010 Workshop, ELRA, Valletta, Malta. https://radimrehurek.com/gensim/lrec2010_final.pdf
[^2]: Gensim documentation, "Introduction / design principles" (memory independence, BLAS delegation). https://radimrehurek.com/gensim/intro.html
[^3]: Gensim README — "Gensim is in stable maintenance mode: we are not accepting new features, but bug and documentation fixes are still welcome." https://github.com/piskvorky/gensim
[^4]: Gensim 4.0.0 migration guide (Python 3 only, KeyedVectors changes, removed modules). https://github.com/piskvorky/gensim/wiki/Migrating-from-Gensim-3.x-to-4

## Tags

python, nlp, topic-modeling, word-embeddings, word2vec, information-retrieval, machine-learning, text-mining, lda, document-similarity, maintenance-mode
