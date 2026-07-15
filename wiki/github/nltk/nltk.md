# nltk/nltk

> The Natural Language Toolkit — a broad, classical NLP library and teaching corpus for Python, predating the deep-learning era.

[GitHub repo](https://github.com/nltk/nltk) ·
[Official website](https://www.nltk.org) ·
[License: Apache-2.0](https://github.com/nltk/nltk/blob/develop/LICENSE.txt)

## Overview

NLTK is a suite of Python modules, data sets, and tutorials for natural language processing. It was started in 2001 by Steven Bird and Edward Loper at the University of Pennsylvania as teaching infrastructure for computational linguistics; the GitHub repository dates to 2009 but the project is older[^1]. Its scope is the classical NLP curriculum: tokenization, part-of-speech tagging, chunking, parsing, stemming and lemmatization, WordNet access, n-gram collocations, naive-Bayes/maxent classification, and a large library of bundled corpora.

The defining tension is **breadth and pedagogy versus production speed**. NLTK is almost entirely pure Python and was designed to be read, inspected, and taught from — every algorithm is legible and swappable. That same design makes it slow and memory-heavy relative to compiled or neural alternatives, and many of its default models are pre-deep-learning (rule-based or linear statistical). For learning what a POS tagger or a CFG parser actually does, NLTK is unmatched; for a high-throughput production pipeline, it is usually the wrong tool.

The companion book, *Natural Language Processing with Python* (Bird, Loper, Klein, O'Reilly 2009), is effectively the library's specification and the reason NLTK remains a fixture in university courses[^2]. As of 2026 the project is still maintained but evolves slowly — bug fixes, Python-version support, and security patches dominate over new capability.

## Getting Started

```bash
pip install nltk
```

Data is **not** bundled with the package — corpora and models are downloaded separately at runtime:

```python
import nltk
nltk.download("punkt_tab")            # sentence/word tokenizer tables
nltk.download("averaged_perceptron_tagger_eng")

from nltk.tokenize import word_tokenize
from nltk import pos_tag

text = "NLTK ships classical NLP algorithms for Python."
tokens = word_tokenize(text)
print(pos_tag(tokens))
# [('NLTK', 'NNP'), ('ships', 'VBZ'), ('classical', 'JJ'), ...]
```

Missing data raises `LookupError` with instructions to run the relevant `nltk.download(...)`. Running `nltk.download()` with no argument opens an interactive downloader UI.

## Architecture / How It Works

NLTK is organized as a flat set of subpackages under `nltk.*`, each covering one classical task:

- **`nltk.tokenize`** — Punkt sentence tokenizer (unsupervised, model-based) plus regex/Treebank word tokenizers.
- **`nltk.tag`** — the averaged perceptron tagger (default, English), plus HMM, Brill, and n-gram taggers you can train yourself.
- **`nltk.corpus`** — lazy corpus readers over the downloaded data: Brown, Gutenberg, Reuters, and the WordNet interface (`nltk.corpus.wordnet`).
- **`nltk.stem`** — Porter, Snowball, and Lancaster stemmers plus the WordNet lemmatizer.
- **`nltk.parse` / `nltk.chunk`** — CFG, chart, dependency, and shallow chunk parsers.
- **`nltk.classify` / `nltk.sentiment`** — naive Bayes, maximum entropy, and the rule-based VADER sentiment analyzer.

The **code/data split** is the central architectural fact. The pip package is small; the actual models and corpora live in separate download packages resolved by `nltk.data.find()` against a search path (`~/nltk_data`, `NLTK_DATA`, and system directories). This keeps installs light and lets corpora carry their own licenses, but it makes NLTK stateful in a way that surprises people: the same code fails or succeeds depending on what has been downloaded on that machine.

Many components are **thin wrappers over external tools** rather than native implementations — historically Stanford CoreNLP, MaltParser, and Megam via subprocess/JVM bridges. These wrappers add setup burden and are the most fragile parts of the library.

## Production Notes

**Data provisioning is the number-one operational footgun.** Because models download at runtime, a container or CI job that works locally will throw `LookupError` in production unless the data is baked into the image (`RUN python -m nltk.downloader punkt_tab ...`) or mounted. Never rely on first-request download in a serving path.

**The `punkt` → `punkt_tab` break.** NLTK 3.9 (2024) changed the tokenizer data package from `punkt` to `punkt_tab` and removed pickle-based loading of several resources in response to a remote-code-execution vulnerability (CVE-2024-39705) affecting the unpickling of downloaded data through 3.8.1[^3]. Code and tutorials that call `nltk.download("punkt")` silently break after upgrading; they must switch to `punkt_tab`. Treat any pinned NLTK below 3.9 as a security liability.

**Performance.** Tokenization and tagging are pure-Python and single-threaded. On large document volumes NLTK is commonly an order of magnitude or more slower than spaCy's Cython pipeline. NLTK has no built-in batching, no GPU path, and its objects (e.g. WordNet synset graphs) can be memory-heavy. Parallelism is your responsibility (multiprocessing over documents).

**Quality and language coverage.** Default taggers and tokenizers are tuned for English; the perceptron tagger's shipped model is English-only. Accuracy on modern, informal, or non-English text lags neural taggers (Stanza, spaCy transformer pipelines). NLTK is best treated as a toolbox of algorithms to train and inspect, not as a source of state-of-the-art pretrained models.

**API stability.** The public API has been stable for years — a strength for long-lived code. The instability is in the data packages and their names, not the function signatures.

## When to Use / When Not

**Use when:**
- Teaching or learning NLP fundamentals — the algorithms are readable and the book is the curriculum.
- You need WordNet, classic corpora, or a specific classical algorithm (Punkt, Brill tagger, chart parser) with minimal ceremony.
- Prototyping and linguistics research where inspectability beats throughput.
- You want to train and compare your own taggers/classifiers on labeled data.

**Avoid when:**
- You need production throughput or low latency on large document volumes.
- You want state-of-the-art accuracy or transformer models out of the box.
- Your workload is heavily non-English.
- You want a single opinionated pipeline object rather than an à la carte toolbox.

## Alternatives

- explosion/spaCy — use instead when you need a fast, production-oriented pipeline (tokenize/tag/parse/NER) with pretrained models.
- stanfordnlp/stanza — use when you need neural accuracy and broad multilingual coverage.
- huggingface/transformers — use when the task calls for transformer models and fine-tuning rather than classical algorithms.
- piskvorky/gensim — use for topic modeling, word2vec, and document similarity rather than linguistic annotation.
- sloria/TextBlob — use when you want a simpler high-level API; it is built on top of NLTK.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2001 | Project started at U. Penn as teaching infrastructure[^1]. |
| 2.0 | 2011 | Rewrite; Python 2 era, expanded corpora and taggers. |
| 3.0 | 2014-09 | Python 3 support, packaging modernization[^2]. |
| 3.5 | 2020-04 | Dropped Python 2. |
| 3.8 | 2023-01 | Maintenance release line. |
| 3.9 | 2024-08 | `punkt_tab`, removal of pickle loading; fixes CVE-2024-39705[^3]. |

## References

[^1]: Steven Bird, Ewan Klein, Edward Loper — *Natural Language Processing with Python*, project background. https://www.nltk.org/
[^2]: Bird, Loper, Klein (2009), *Natural Language Processing with Python*, O'Reilly Media. https://www.nltk.org/book/
[^3]: CVE-2024-39705 — deserialization of untrusted data in NLTK ≤ 3.8.1; fixed in 3.9. https://nvd.nist.gov/vuln/detail/CVE-2024-39705

## Tags

python, nlp, natural-language-processing, tokenization, pos-tagging, wordnet, computational-linguistics, machine-learning, text-processing, library
