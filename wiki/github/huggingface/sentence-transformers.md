# huggingface/sentence-transformers

> Python framework for computing, training, and serving dense, sparse, and cross-encoder text embeddings — the reference implementation of SBERT.

[GitHub repo](https://github.com/huggingface/sentence-transformers) ·
[Official website](https://www.sbert.net) ·
[License: Apache-2.0](https://github.com/huggingface/sentence-transformers/blob/main/LICENSE)

## Overview

Sentence Transformers (a.k.a. SBERT) is the de facto standard library for turning text into fixed-length embedding vectors in Python. It began as the code accompanying the 2019 EMNLP paper "Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks" by Nils Reimers and Iryna Gurevych at the UKP Lab, TU Darmstadt[^1]. The insight of that paper is what the library still encapsulates: a raw BERT produces token embeddings that are poor for similarity out of the box, but a network fine-tuned with a siamese/triplet objective and a pooling layer produces sentence vectors whose cosine similarity is semantically meaningful and cheap to compute at scale.

The project was originally hosted under `UKPLab/sentence-transformers`; it has since moved to the `huggingface` organization and is maintained by Tom Aarsen at Hugging Face (the old URL still redirects). It is the primary consumer path for the 15,000+ embedding models tagged `sentence-transformers` on the Hugging Face Hub, and it sits underneath most Python RAG stacks — LangChain, LlamaIndex, Haystack, and Chroma all default to it for local embedding.

Three model families now live under one API: `SentenceTransformer` (dense bi-encoders), `CrossEncoder` (rerankers that score a query–document pair jointly), and, since v5, `SparseEncoder` (SPLADE-style sparse lexical embeddings)[^2]. The defining tension of the library is scope creep versus stability: it has grown from "load a model and call `.encode()`" into a full training framework, and the training API was rewritten twice in two years (v3, v4), which is a real migration burden for anyone who wrote fine-tuning code against the old `model.fit()`.

## Getting Started

```bash
pip install -U sentence-transformers
```

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("sentence-transformers/all-MiniLM-L6-v2")

sentences = [
    "The weather is lovely today.",
    "It's so sunny outside!",
    "He drove to the stadium.",
]
embeddings = model.encode(sentences)   # (3, 384) numpy array
similarities = model.similarity(embeddings, embeddings)
print(similarities)
# tensor([[1.0000, 0.6660, 0.1046],
#         [0.6660, 1.0000, 0.1411],
#         [0.1046, 0.1411, 1.0000]])
```

Retrieve-and-rerank, the canonical two-stage pattern — a fast bi-encoder narrows the candidate set, a slower cross-encoder reorders the top-k:

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L6-v2")
ranks = reranker.rank("How many people live in Berlin?", passages, return_documents=True)
```

## Architecture / How It Works

A `SentenceTransformer` is an ordered list of modules, serialized as `modules.json` in the model directory. The typical stack is `Transformer` (a Hugging Face `transformers` backbone) → `Pooling` (mean/CLS/max over token embeddings) → optional `Dense`/`Normalize`. `model.encode()` tokenizes, runs a forward pass, pools, and returns vectors. Because the modules are just serialized config, a model on the Hub is portable across the whole ecosystem without custom code.

The **bi-encoder vs cross-encoder** distinction is the core architectural idea and the most common source of confusion. A bi-encoder embeds each text independently, so N documents = N vectors you can index once and reuse; similarity is a cheap dot product. A cross-encoder feeds the (query, document) pair through the model together and outputs a single relevance score — more accurate, but it cannot be precomputed, so scoring N documents means N forward passes per query. You use bi-encoders to retrieve and cross-encoders to rerank; you never use a cross-encoder as your primary index.

`SparseEncoder` (v5) produces high-dimensional sparse vectors indexed by vocabulary token (e.g. 30k dims, ~99.8% zeros), bridging neural embeddings and classic inverted-index / BM25-style retrieval for hybrid search.

Training was rebuilt around the Hugging Face `Trainer`. As of v3+, `SentenceTransformerTrainer` takes a `datasets.Dataset`, a loss module (20+ are provided — MultipleNegativesRankingLoss, CoSENTLoss, triplet, contrastive, etc.), and an evaluator, mirroring the `transformers` training loop rather than the old bespoke `model.fit()`. v4 did the same rewrite for `CrossEncoderTrainer`. Inference backends beyond PyTorch — ONNX and OpenVINO — are selectable via the `backend=` argument for faster CPU serving.

## Production Notes

**Silent truncation.** Every model has a `max_seq_length` (often 256 or 512 tokens depending on the checkpoint). Text longer than that is truncated with no warning, so long documents get embedded from only their first chunk. Check `model.max_seq_length` and chunk deliberately; do not assume your 2,000-token document was fully represented.

**Instruction prompts are model-specific and easy to miss.** Many strong models (E5, BGE, GTE, and others) require a prefix such as `"query: "` / `"passage: "`, and asymmetric models expect different prefixes for queries and documents. Omitting them, or applying the same prefix to both sides, quietly tanks retrieval quality. Use the `prompt`/`prompt_name` arguments and read the model card — nothing enforces this for you.

**Normalize before dot-product indexing.** Vector databases configured for cosine or inner-product similarity assume unit-length vectors. Pass `normalize_embeddings=True` (or use `similarity_fn_name`) so your index math matches how the model was trained; a mismatch degrades ranking silently.

**Throughput.** `encode()` batches internally (`batch_size`, default 32); tune it to your GPU memory. For large corpora, `encode_multi_process()` / `start_multi_process_pool()` shards across GPUs or CPU workers. For CPU-only deployment, the ONNX/OpenVINO backends, quantization (binary/scalar), Matryoshka truncation, and static-embedding models are the levers — a full transformer on CPU is slow.

**Version churn.** The v2 → v3 jump deprecated the old training path in favor of the `Trainer`; `model.fit()` still exists but is soft-deprecated, and tutorials written against it no longer reflect the recommended API. v4 (CrossEncoder) and v5 (SparseEncoder) each added a parallel training stack. Pin your version if you have working fine-tuning code, and expect docs found via search to lag the installed release. Also mind the peer dependency on `transformers`: an ST upgrade can pull a `transformers` bump that touches the rest of your stack.

**Serving.** The library is a Python in-process embedder, not a server. For high-throughput or multi-tenant serving, put the model behind a dedicated inference server rather than calling `encode()` inside a request handler.

## When to Use / When Not

**Use when:**
- You need local, open-weight embeddings for semantic search, clustering, RAG, or dedup without sending text to a hosted API.
- You want to run models from the MTEB leaderboard with a one-line load.
- You need to fine-tune an embedding or reranker on your own domain data.
- You want retrieve-and-rerank in one library.

**Avoid when:**
- You only need embeddings occasionally and a hosted API (OpenAI, Cohere, Voyage) is cheaper than operating a model.
- You need a production-grade, language-agnostic embedding server at scale — reach for a dedicated serving stack.
- You need ultra-low-latency CPU embeddings and can accept lower quality — static/word-vector approaches are far faster.
- You need tight control over the raw forward pass; drop down to `transformers` directly.

## Alternatives

- huggingface/text-embeddings-inference — use when you need to *serve* embeddings and rerankers as a fast Rust HTTP service instead of embedding in a Python process.
- FlagOpen/FlagEmbedding — use when you want the BGE model family with its own training/fine-tuning recipes and reranker tooling.
- huggingface/transformers — use when you need lower-level control over tokenization and the forward pass, foregoing the pooling/encode convenience.
- MinishLab/model2vec — use when you need extremely fast CPU-only static embeddings and can trade accuracy for speed.
- neuml/txtai — use when you want an all-in-one embeddings database plus pipelines rather than just the embedding layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.2 | 2019-08 | First public release alongside the SBERT paper[^1]. |
| 1.0 | 2021-10 | API stabilization; broader model/loss support. |
| 2.0 | 2021-10 | Hub integration; large expansion of pretrained models. |
| 3.0 | 2024-05 | Training rewritten around the HF `Trainer`; `datasets` + multi-loss[^2]. |
| 4.0 | 2025-03 | CrossEncoder (reranker) training refactor to match v3. |
| 5.0 | 2025-06 | SparseEncoder / SPLADE support; ONNX & OpenVINO backends matured. |

## References

[^1]: Nils Reimers and Iryna Gurevych, "Sentence-BERT: Sentence Embeddings using Siamese BERT-Networks", EMNLP 2019. https://arxiv.org/abs/1908.10084
[^2]: Sentence Transformers documentation (SBERT.net) — quickstart and training overviews for Sentence Transformer, Cross Encoder, and Sparse Encoder models. https://www.sbert.net/docs/quickstart.html

## Tags

python, embeddings, sentence-embeddings, semantic-search, nlp, information-retrieval, reranking, transformers, sbert, rag, huggingface, machine-learning
