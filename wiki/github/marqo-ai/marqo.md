# marqo-ai/marqo

> A batteries-included vector search engine that generates the embeddings for you — now deprecated as open source in favor of a commercial ecommerce product.

[GitHub repo](https://github.com/marqo-ai/marqo) ·
[Official website](https://www.marqo.ai/) ·
[License: Apache-2.0](https://github.com/marqo-ai/marqo/blob/mainline/LICENSE)

## Overview

Marqo is an end-to-end tensor (vector) search engine. Unlike a pure vector database, it bundles the embedding step: you send raw text or images, and Marqo chunks them, runs them through a model (sentence-transformers, OpenCLIP, ONNX, or custom), stores the resulting vectors, and serves approximate-nearest-neighbor queries — all behind one HTTP API and Python client[^1]. The pitch was that "bring your own embeddings" systems (Pinecone, Weaviate, Qdrant) push inference, chunking, and model management onto the application, and Marqo folds that into the engine.

The project is built and maintained by Marqo (an Australian company, formerly S2Search). It started in 2022 as a general-purpose multimodal search engine and gained traction for image+text CLIP search working out of the box. Over 2024–2026 the company narrowed its commercial focus to ecommerce search and personalization, and in 2026 marked the open-source project **deprecated** — the README now states it "will no longer receive updates" and directs users to the hosted product[^2]. The repository is not yet archived and the last tagged release was 2.26.0 (April 2026), but no further development is expected.

The defining tension: convenience versus control. Bundling inference makes the "hello world" trivial but couples model serving, GPU scheduling, and vector storage into a single deployment unit that is harder to scale independently than a decoupled embed-then-store pipeline.

## Getting Started

Marqo runs as a Docker container that bundles the API, the inference layer, and the storage backend:

```bash
docker pull marqoai/marqo:latest
docker run --name marqo -p 8882:8882 marqoai/marqo:latest
pip install marqo
```

```python
import marqo

mq = marqo.Client(url="http://localhost:8882")
mq.create_index("products")

mq.index("products").add_documents(
    [
        {"title": "Running shoes", "desc": "Lightweight trail runners"},
        {"title": "Wool socks",    "desc": "Warm merino hiking socks"},
    ],
    tensor_fields=["title", "desc"],   # fields to embed; others stay lexical
)

results = mq.index("products").search("footwear for the mountains")
print(results["hits"][0])
```

The same API accepts image URLs in documents when the index is created with a CLIP-family model, enabling cross-modal (text-query → image-result) retrieval without a separate pipeline.

## Architecture / How It Works

Marqo is three concerns in one box: an **API/orchestration layer**, an **inference layer** (model download, batching, CPU/GPU execution), and a **storage/retrieval backend**.

The backend has changed once, significantly. Marqo 1.x stored and searched vectors in **Marqo-OS**, a fork of OpenSearch. Marqo 2.0 replaced that with **Vespa** as the underlying index and query engine[^3] — visible in the current source, where index management compiles Marqo index definitions into Vespa schema documents (`.sd` files) and query logic runs through a Vespa application package. This migration changed on-disk format and forced a reindex; it is the single largest architectural discontinuity in the project's history.

Key concepts:

- **Tensor fields** — fields you explicitly nominate (`tensor_fields=[...]`) get chunked and embedded; everything else is stored as filterable/lexical metadata. Earlier versions used the inverse `non_tensor_fields` flag.
- **Structured vs. unstructured indexes** — later 2.x adds "structured" indexes with declared schemas (faster, stricter) alongside the original schemaless "unstructured" mode. The source also carries a "semi-structured" index path bridging the two.
- **Hybrid search** — combines tensor (semantic) scoring with BM25 lexical scoring, reconciled by the Vespa layer.
- **Multimodal combination fields** — one vector built from a weighted blend of several sub-fields (e.g. image 0.8 + caption 0.2).
- **Inference** — models run in-process by default. GPU is used automatically inside the CUDA image; the CPU image is convenient for local dev but slow for large batches.

## Production Notes

**"Deprecated but not archived" is the headline caveat.** New adopters are building on a codebase the vendor has stated will not be updated. Security patches, model additions, and Vespa-version bumps should be assumed to stop. Evaluate accordingly before putting it on a critical path.

**The bundled container is a monolith.** API, inference, and Vespa run together, which is friendly for a laptop and awkward for production. Scaling inference (GPU-bound) and storage (memory/disk-bound) independently means moving off the single-container setup toward a distributed deployment — historically the least-documented part of the system, and typically the point where teams engaged the commercial offering.

**Memory and GPU.** HNSW-style indexes are memory-resident; large corpora need substantial RAM on the storage nodes. The default CPU image will bottleneck on embedding throughput during bulk ingestion — a first indexing run of millions of documents is inference-bound, not I/O-bound. Use the CUDA image and tune batch size for any non-trivial dataset.

**Reindexing is the recurring pain.** Changing the embedding model, chunking strategy, or (historically) crossing the 1.x→2.x backend change all require rebuilding the index from source documents, because the stored vectors are model-specific. Keep the original documents; the index is not the source of truth.

**Version coupling.** Because Marqo pins a specific Vespa version and a specific set of model runtimes inside the image, you upgrade the whole container, not parts of it. Read release notes for backend-format changes before bumping across minor versions.

## When to Use / When Not

**Use when:**
- You want semantic or multimodal (image+text) search working quickly without assembling an embed→store→query pipeline yourself.
- Your team lacks ML-serving infrastructure and prefers the engine to own model execution.
- You are prototyping and value one Docker command over integrating a separate model server and vector DB.

**Avoid when:**
- You need a supported, actively maintained OSS dependency — the project is deprecated; this is disqualifying for many.
- You already run your own embedding pipeline and just need a vector store — a decoupled DB (Qdrant, Milvus) gives more control.
- You need to scale inference and storage independently at large volume, where the bundled architecture fights you.
- You require the raw query/index tuning surface of the underlying engine — Vespa used directly exposes far more.

## Alternatives

- weaviate/weaviate — vector DB with optional built-in vectorizer modules; closest "batteries-included" analogue with an active project and managed cloud.
- qdrant/qdrant — Rust vector database, bring-your-own-embeddings; use when you want a fast, well-maintained store and already produce vectors.
- milvus-io/milvus — distributed vector database built for billion-scale corpora; use when storage scale is the primary constraint.
- vespa-engine/vespa — the engine underneath Marqo 2.x; use directly when you want full control over ranking, schemas, and scaling and can absorb the operational complexity.
- elastic/elasticsearch — mature lexical + dense-vector hybrid search; use when you also need full-text, aggregations, and an established operational story.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2022-08 | Repository created; general multimodal tensor search on the Marqo-OS (OpenSearch fork) backend[^1]. |
| 2.x | 2024 | Backend migrated to Vespa; `tensor_fields`, structured indexes, hybrid search[^3]. |
| 2.5–2.24 | 2024–2026 | Steady minor releases: model additions, structured/semi-structured indexes, performance work. |
| 2.26.0 | 2026-04-07 | Last tagged release. |
| deprecated | 2026 | README marks OSS deprecated; users directed to the hosted ecommerce product[^2]. |

## References

[^1]: Marqo documentation. https://docs.marqo.ai
[^2]: marqo-ai/marqo README, deprecation notice (retrieved 2026-07). https://github.com/marqo-ai/marqo
[^3]: Vespa backend, evidenced by `core/index_management/vespa_application_package.py` and Vespa schema templates (`.sd.jinja2`) in the repository source. https://github.com/marqo-ai/marqo/tree/mainline/components/marqo/src/marqo/core

## Tags

vector-search, semantic-search, embeddings, multimodal, machine-learning, search-engine, python, vespa, ecommerce, deprecated
