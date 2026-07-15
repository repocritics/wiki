# MaartenGr/BERTopic

> Topic modeling as a swappable pipeline of embeddings, dimensionality reduction, clustering, and a class-based TF-IDF — not a probabilistic generative model.

[GitHub repo](https://github.com/MaartenGr/BERTopic) ·
[Official website](https://maartengr.github.io/BERTopic/) ·
[License: MIT](https://github.com/MaartenGr/BERTopic/blob/master/LICENSE)

## Overview

BERTopic is a Python topic-modeling library by Maarten Grootendorst, first released in 2020 and described in a 2022 paper[^1]. It reframes topic modeling as a modular clustering pipeline rather than a probabilistic generative process: documents are embedded, the embeddings are reduced in dimensionality, the reduced vectors are clustered, and each cluster is then described by a class-based TF-IDF (c-TF-IDF) that treats every cluster as one long document and surfaces its most distinctive terms.

The defining tradeoff is that BERTopic is not LDA. Classic topic models (LDA, NMF) are generative and assign every document a distribution over topics. BERTopic, in its default configuration, does hard clustering with HDBSCAN: each document lands in exactly one topic, and documents that fit no cluster are assigned to topic `-1`, the outlier bucket. This makes topics more interpretable and embedding-quality-driven, but it also means "how many documents are outliers" and "how many topics exist" are emergent properties of clustering hyperparameters, not knobs you set directly.

The audience is data scientists and NLP practitioners who want interpretable topics over medium-sized corpora, plus teams layering LLMs on top to auto-label clusters. Its reach is broad: the same object exposes guided, supervised, semi-supervised, dynamic (over time), hierarchical, class-based, online/incremental, zero-shot, and multimodal variants, all through one `BERTopic` class[^2].

## Getting Started

```bash
pip install bertopic
# extras: embedding backends and image support
pip install bertopic[flair,gensim,spacy,use]
pip install bertopic[vision]
```

```python
from bertopic import BERTopic
from sklearn.datasets import fetch_20newsgroups

docs = fetch_20newsgroups(subset="all", remove=("headers", "footers", "quotes"))["data"]

topic_model = BERTopic()
topics, probs = topic_model.fit_transform(docs)

topic_model.get_topic_info()   # topic ids, counts, auto labels (-1 = outliers)
topic_model.get_topic(0)       # top terms + c-TF-IDF weights for topic 0
```

## Architecture / How It Works

BERTopic's core is a six-stage pipeline where every stage is a replaceable component[^3]:

1. **Embed** — sentence-transformers by default (`all-MiniLM-L6-v2`). Swappable for Flair, spaCy, USE, OpenAI, Cohere, or precomputed embeddings.
2. **Reduce dimensionality** — UMAP by default, to make clustering tractable in low-dimensional space.
3. **Cluster** — HDBSCAN by default, which is density-based and produces the `-1` outlier class.
4. **Tokenize** — CountVectorizer, applied per-cluster.
5. **Weight** — c-TF-IDF: concatenate all documents in a cluster into one pseudo-document, then compute a modified TF-IDF so terms distinctive to a cluster score highest.
6. **Represent** — optional fine-tuning of the term list via `KeyBERTInspired`, MMR, part-of-speech filters, or an LLM (OpenAI, LangChain, local models) that rewrites the top terms into labels or summaries.

The insight that makes the library cohere is that clustering and representation are decoupled. Because the top-term extraction (c-TF-IDF) is a separate post-hoc step, you can re-run `.update_topics()` with different n-gram ranges or representation models without re-embedding or re-clustering. It also means the "topics" are labels attached to clusters after the fact, not latent variables the model inferred jointly.

This modularity is genuine — you can substitute cuML's GPU UMAP/HDBSCAN, replace HDBSCAN with k-means for a fixed topic count, or skip dimensionality reduction entirely. But the stages are not fully independent in practice: UMAP's output geometry strongly shapes what HDBSCAN finds, so tuning `n_neighbors`, `min_dist`, and `min_cluster_size` together is where most real-world effort goes.

## Production Notes

- **Non-determinism by default.** UMAP is stochastic; two runs on identical data yield different topics unless you pass `umap_model=UMAP(random_state=42)`. Setting a seed also disables UMAP's parallelism, so reproducibility costs wall-clock time.
- **`.transform()` is approximate.** Predicting topics for new documents projects them through the fitted UMAP and does an approximate-predict against HDBSCAN. Results for unseen data are noisier than the training-time assignments, and probabilities are only produced when HDBSCAN is the clusterer.
- **Scale ceiling.** The default UMAP + HDBSCAN path is memory-hungry and slows sharply past a few hundred thousand documents. For millions of docs the practical routes are GPU acceleration via cuML, or swapping in k-means / online clustering. There is no free lunch: the interpretability comes from clustering that does not scale linearly.
- **Outliers are a tuning problem.** A large `-1` topic is common and expected. `reduce_outliers()` reassigns them post-hoc using c-TF-IDF, embeddings, or distributions, but this is a separate reconciliation step that can change downstream counts.
- **Saving models has three formats.** `safetensors` (recommended) and `pytorch` store only the topic data and reference the embedding model by name — you must have the same sentence-transformer available at load time. Pickle serializes everything but is version-fragile and unsafe to load from untrusted sources. Loading a `safetensors` model on a machine without the original embedding model will fail or silently degrade `.transform()`.
- **Version churn in dependencies.** BERTopic tracks fast-moving libraries (sentence-transformers, UMAP, HDBSCAN, numba). Environment breakage usually originates in a transitive dependency (numba vs numpy versions) rather than BERTopic itself; pin the stack.

## When to Use / When Not

**Use when:**
- You have short-to-medium documents and want interpretable, embedding-driven topics.
- You want to swap embedding/clustering components or layer an LLM to auto-label clusters.
- You need dynamic, hierarchical, guided, or per-class topic variants from one API.

**Avoid when:**
- You need a true generative model with per-document topic distributions (LDA/NMF fit that better).
- You are modeling tens of millions of documents on CPU and cannot use GPU clustering.
- You require deterministic, reproducible topics with zero configuration.
- Your corpus is tiny (hundreds of docs): density clustering will dump most of it into `-1`.

## Alternatives

- MaartenGr/KeyBERT — same author; use when you want keyword/keyphrase extraction per document rather than corpus-level topics.
- ddangelov/Top2Vec — similar embed-then-cluster philosophy; use when you want a more all-in-one design with fewer swappable parts.
- scikit-learn (LatentDirichletAllocation / NMF) — use when you need a classic probabilistic model, per-document topic mixtures, or minimal dependencies.
- piskvorky/gensim — use for LDA/NMF at scale, streaming corpora, and word2vec-style tooling on modest compute.
- scikit-learn KMeans + TF-IDF — use when you want a transparent, fully deterministic baseline before reaching for embeddings.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2020-10 | Embedding + clustering + c-TF-IDF pipeline. |
| Paper | 2022-03 | "Neural topic modeling with a class-based TF-IDF procedure"[^1]. |
| Repo created | 2020-09-22 | First public commit on GitHub[^4]. |

## References

[^1]: Maarten Grootendorst, "BERTopic: Neural topic modeling with a class-based TF-IDF procedure," arXiv:2203.05794, 2022. https://arxiv.org/abs/2203.05794
[^2]: BERTopic documentation — algorithm and variations. https://maartengr.github.io/BERTopic/
[^3]: BERTopic documentation — "The Algorithm" (modular six-step pipeline). https://maartengr.github.io/BERTopic/algorithm/algorithm.html
[^4]: GitHub repository metadata (created 2020-09-22, MIT license). https://github.com/MaartenGr/BERTopic

## Tags

python, nlp, topic-modeling, clustering, sentence-embeddings, transformers, umap, hdbscan, c-tf-idf, unsupervised-learning, machine-learning
