# facebookresearch/fastText

> A CPU-only C++ library for word embeddings and linear text classification, built around character n-grams — fast, small, and now archived.

[GitHub repo](https://github.com/facebookresearch/fastText) ·
[Official website](https://fasttext.cc/) ·
[License: MIT](https://github.com/facebookresearch/fastText/blob/main/LICENSE)

## Overview

fastText is a library from Facebook AI Research (FAIR) for two related tasks: learning
word representations and training supervised text classifiers. It was released in 2016
alongside two papers — "Enriching Word Vectors with Subword Information" and "Bag of
Tricks for Efficient Text Classification"[^1][^2]. Its distinguishing idea is treating each
word as a bag of character n-grams rather than an atomic token, so a word vector is the sum
of its subword vectors. This gives it morphology awareness and the ability to produce
vectors for out-of-vocabulary words — something the original word2vec cannot do.

The defining tension is era. fastText belongs to the pre-transformer embedding generation:
it is a shallow linear model, not a contextual one. It produces a single static vector per
word regardless of surrounding context, so "bank" (river) and "bank" (money) share one
embedding. In exchange it is extraordinarily cheap — it trains on a billion words in
minutes on a laptop CPU, runs inference without a GPU, and ships models that can be
quantized to under a megabyte. For high-throughput, low-cost, on-CPU classification and
language identification it remains competitive in 2026; for anything needing semantic
context, it has been superseded by transformer embeddings.

The repository was **archived by its maintainers in March 2024**[^3]. It is read-only: no
new releases, bug fixes, or security patches. The last tagged release is v0.9.2 (April
2020)[^4]. It is still very widely deployed, but treat it as frozen, mature software rather
than an active project.

## Getting Started

```bash
# Python (most common path today)
pip install fasttext

# Or build the CLI from source
git clone https://github.com/facebookresearch/fastText.git
cd fastText && make          # produces the ./fasttext binary
```

```python
import fasttext

# Supervised text classification — labels prefixed with __label__ in the training file
model = fasttext.train_supervised("train.txt", epoch=25, lr=1.0, wordNgrams=2)
print(model.predict("which baking dish is best to bake a banana bread"))
# (('__label__baking',), array([0.98]))

model.save_model("model.bin")
model.quantize(input="train.txt", retrain=True)   # shrink to a .ftz
```

```bash
# CLI equivalents
$ ./fasttext supervised -input train.txt -output model -epoch 25 -lr 1.0 -wordNgrams 2
$ ./fasttext test model.bin test.txt        # prints P@1 and R@1
$ ./fasttext skipgram -input data.txt -output vecs   # unsupervised word vectors
```

## Architecture / How It Works

fastText is a single-machine, multithreaded C++ program with a thin pybind11 wrapper. There
is no distributed training and no GPU code path — all math is CPU SIMD-friendly float
operations, parallelized across threads with Hogwild-style lock-free asynchronous SGD.

**Subword model.** Every word is decomposed into character n-grams of length `minn`–`maxn`
(default 3–6 for unsupervised modes) plus the whole word. Each n-gram has its own vector;
the word vector is their sum. To keep memory bounded, n-grams are hashed into a fixed number
of buckets (`-bucket`, default 2,000,000) — a collision-tolerant hashing trick. This is what
lets fastText emit a plausible vector for a word it never saw during training, and why it
handles morphologically rich languages (Finnish, Turkish, Korean) better than word2vec.

**Two training objectives share one codebase.** Unsupervised `skipgram`/`cbow` learn word
vectors from raw text. Supervised classification averages the embeddings of a sentence's
words and n-grams into a single hidden vector, then applies a linear classifier. The
"efficiency" of the classification paper comes from this linearity plus a **hierarchical
softmax** (a Huffman tree over labels) that turns an O(K) output layer into O(log K) — the
practical enabler when you have thousands of labels. Loss can also be negative sampling or
full softmax (`-loss {ns, hs, softmax}`).

**Quantization.** The `quantize` command applies product quantization (from the FastText.zip
work[^5]) to compress a trained supervised model into a `.ftz`, often reducing size by
10–100× with small accuracy loss. This is the mechanism behind the sub-megabyte pretrained
classifiers.

The output format is two files: `model.bin` (full parameters, dictionary, hyperparameters —
reloadable) and `model.vec` (plain-text vectors, one word per line). Nearly the entire
public surface is these formats plus a handful of CLI verbs; the library does not evolve an
API the way a framework does.

## Production Notes

**Archived means frozen.** No upstream fixes since March 2024. The biggest practical
consequence: the official `fasttext` PyPI package does not build cleanly against newer
Python/compiler combinations, and many teams have moved to the community `fasttext-wheel`
package for prebuilt binaries. Pin your toolchain or use a wheel; do not expect `pip install
fasttext` to "just work" on the newest Python.

**Model size is the recurring surprise.** The pretrained 157-language `cc.*.300.bin` vectors
are large — several gigabytes each uncompressed — because they carry full n-gram tables. In
memory-constrained serving, use the `.vec` text vectors or quantized `.ftz` models instead
of the `.bin`. Training supervised models with `-wordNgrams 2` and a large `-bucket` can also
consume many gigabytes of RAM; reduce `-bucket` or `-dim` if you OOM.

**Language identification is the single most-deployed artifact.** The `lid.176` model
identifies 176 languages, ships as a ~126 MB `.bin` or a ~917 KB `.ftz`, and is used
throughout the NLP ecosystem (CommonCrawl filtering, dataset language tagging). It is fast
and good, but it classifies by character/word statistics, so very short strings, code-mixed
text, and romanized non-Latin languages degrade noticeably.

**No context, by design.** Because embeddings are static, fastText is a poor fit anywhere
word sense disambiguation matters. Teams frequently mistake it for a drop-in replacement for
sentence-transformers and get worse retrieval quality; it is not the same class of model.

**Determinism and threads.** Hogwild async SGD means training is not bit-reproducible across
thread counts; set `-thread 1` if you need reproducibility, at a large speed cost. Results
also swing with `-lr`, `-epoch`, and `-wordNgrams`, which are under-tuned by default for
classification (the defaults favor speed, not accuracy).

## When to Use / When Not

**Use when:**
- You need fast, cheap text classification or language ID on CPU at high volume.
- You want embeddings that handle OOV / misspelled / morphologically rich words.
- You need tiny, quantizable models for edge or embedded deployment.
- You are reproducing or building on the fastText papers, or need the pretrained 157-language vectors.

**Avoid when:**
- You need context-sensitive or sentence-level semantics — use transformer embeddings.
- You want an actively maintained dependency with security updates (it is archived).
- Your task is dominated by word sense or long-range meaning rather than surface features.
- You need GPU-scale training of large models — fastText is CPU-only and shallow by design.

## Alternatives

- piskvorky/gensim — Python-native word2vec and a fastText re-implementation; better for scripted training and analysis pipelines when you want to stay in Python.
- explosion/spaCy — use instead when you want a maintained, production NLP pipeline (tagging, NER, vectors) rather than a raw embedding trainer.
- huggingface/transformers — use when you need contextual embeddings or classification accuracy over speed and cost.
- UKPLab/sentence-transformers — use for semantic search / retrieval where sentence-level meaning matters.
- facebookresearch/StarSpace — the same lab's general-purpose embedding model for entities beyond words (documents, users, graphs).

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-08 | Open-sourced with the subword and classification papers[^1][^2]. |
| pretrained vectors | 2017–2018 | Wikipedia + Common Crawl vectors for 157 languages released[^6]. |
| v0.9.1 | 2019-05 | First official Python bindings via pybind11. |
| v0.9.2 | 2020-04 | Latest release; bug fixes, autotune improvements[^4]. |
| archived | 2024-03 | Repository set read-only; development ended[^3]. |

## References

[^1]: P. Bojanowski, E. Grave, A. Joulin, T. Mikolov, "Enriching Word Vectors with Subword Information," TACL 2017. https://arxiv.org/abs/1607.04606
[^2]: A. Joulin, E. Grave, P. Bojanowski, T. Mikolov, "Bag of Tricks for Efficient Text Classification," EACL 2017. https://arxiv.org/abs/1607.01759
[^3]: Repository archived status via GitHub API (`archived: true`, last push 2024-03-22). https://github.com/facebookresearch/fastText
[^4]: fastText releases — v0.9.2. https://github.com/facebookresearch/fastText/releases
[^5]: A. Joulin et al., "FastText.zip: Compressing text classification models," 2016. https://arxiv.org/abs/1612.03651
[^6]: Word vectors for 157 languages. https://fasttext.cc/docs/en/crawl-vectors.html

## Tags

nlp, word-embeddings, text-classification, machine-learning, cpp, python, subword, language-identification, archived, static-embeddings
