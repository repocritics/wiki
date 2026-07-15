# google/sentencepiece

> Language-independent subword tokenizer that trains BPE or unigram vocabularies directly from raw text, with lossless detokenization baked into a self-contained model file.

[GitHub repo](https://github.com/google/sentencepiece) ·
[License: Apache-2.0](https://github.com/google/sentencepiece/blob/master/LICENSE)

## Overview

SentencePiece is a C++ subword tokenizer and detokenizer, with Python bindings, written primarily by Taku Kudo at Google and first published alongside an EMNLP 2018 system-demo paper[^1]. It solves a narrow but load-bearing problem in NLP pipelines: turning raw text into a fixed-size vocabulary of integer IDs, and back, in a way that is reproducible and language-agnostic. Despite the `google/` namespace, the README states plainly that it "is not an official Google product."

Its defining design choice is to treat input as a raw stream of Unicode characters rather than pre-tokenized words. Whitespace is escaped as a meta symbol `▁` (U+2581) and folded into the vocabulary, so detokenization is a plain string join with no language-specific rules — the same model behaves identically on English, Chinese, Japanese, or Thai, none of which it treats specially. This is what made it the tokenizer of choice for early multilingual models and, later, for a large fraction of open LLMs.

The central tension is that SentencePiece is both ubiquitous and effectively frozen as an interface. Its `.model` protobuf format encodes not just the vocabulary but the normalization rules and segmentation algorithm, so a model file is a versioned contract that downstream frameworks must reproduce byte-for-byte. That guarantee is the source of both its reliability and its most common integration bugs.

## Getting Started

```bash
pip install sentencepiece
```

```python
import sentencepiece as spm

# Train directly from raw text — no pre-tokenization step.
spm.SentencePieceTrainer.train(
    input='corpus.txt', model_prefix='m', vocab_size=8000,
    model_type='unigram',        # or 'bpe', 'char', 'word'
    byte_fallback=True,          # unknown chars -> UTF-8 byte tokens
)

sp = spm.SentencePieceProcessor(model_file='m.model')

ids = sp.encode("I saw a girl with a telescope.", out_type=int)
assert sp.decode(ids) == "I saw a girl with a telescope."   # lossless
```

The trained `m.model` (and companion `m.vocab`) is the artifact you ship. The command-line tools `spm_train` / `spm_encode` / `spm_decode` from the C++ build do the same work outside Python.

## Architecture / How It Works

SentencePiece bundles four segmentation algorithms behind one interface, selected at train time via `model_type`: **unigram** (the default, a probabilistic language-model-based vocabulary from Kudo 2018[^2]), **bpe** (Byte-Pair Encoding, Sennrich et al. 2016[^3]), **char**, and **word**. Unigram and BPE are the ones that matter; they produce meaningfully different segmentations from the same corpus.

Training runs in a single process. It builds a large seed vocabulary of candidate pieces, then — for unigram — iteratively prunes it via EM to maximize corpus likelihood under the fixed target `vocab_size`. This is CPU- and memory-bound and does not distribute across machines; for large corpora you subsample with `input_sentence_size` and `shuffle_input_sentence` rather than feeding everything.

The output `.model` file is a protobuf (`ModelProto`) that embeds the piece-to-ID table, per-piece scores, control/user-defined symbols, and the normalization spec (default `nmt_nfkc`). Because normalization is stored inside the model, encoding is deterministic and portable: the same file yields identical IDs from the C++, Python, or other-language bindings. The processor loads this once and is safe to call concurrently for encoding.

Two later additions are important for LLM use. `byte_fallback` decomposes any out-of-vocabulary character into 256 reserved UTF-8 byte tokens, guaranteeing zero true unknowns — this is why LLaMA-family tokenizers never emit `<unk>`. Subword regularization (unigram sampling) and BPE-dropout let training and inference sample alternative segmentations of the same text as a data-augmentation and robustness mechanism, controlled by `enable_sampling`, `alpha`, and `nbest_size`.

## Production Notes

- **The `.model` file is the contract, not the code version.** Two SentencePiece releases will produce identical IDs from the same model file; that is the whole point. Check the model file into your artifact store and treat it as immutable — retraining with a different `vocab_size` or normalization silently shifts every ID.
- **HuggingFace "fast" conversions are the classic footgun.** `transformers` converts SentencePiece models into its own Rust `tokenizers` format for the fast path. The slow (`use_fast=False`) and fast tokenizers have historically disagreed on edge cases — leading spaces, special-token handling, `add_dummy_prefix`. If encodings mismatch between environments, suspect the slow/fast split first.
- **The leading-space behavior surprises people.** By default (`add_dummy_prefix=true`) SentencePiece prepends `▁` to the input, so `"Hello"` and `" Hello"` can tokenize the same, and the first token of a decode may carry an implicit space. This is the root of a well-known class of LLaMA prompt-templating bugs.
- **Normalization is baked in and not obvious.** The default NFKC-based normalization rewrites some characters before segmentation. For code, math, or exact-string tasks, train with `normalization_rule_name=identity` or a custom rule, or the model will alter input you expected to survive verbatim.
- **Training is single-node and memory-hungry.** Expect to subsample very large corpora. There is no built-in distributed trainer.
- **Vocabulary is fixed at train time.** Adding tokens after the fact means retraining or using `user_defined_symbols` / control symbols reserved up front. You cannot grow the vocabulary of a shipped model without breaking existing IDs.

## When to Use / When Not

**Use when:**
- You are training a model from scratch and need a reproducible, language-independent vocabulary with lossless round-trip.
- You work with CJK / Thai / no-whitespace languages, or truly multilingual corpora.
- You need the exact tokenization of an existing model that ships a `.model` file (LLaMA, T5, ALBERT, XLNet, mBART, Gemma, and many others).

**Avoid when:**
- You only need to tokenize for GPT-family models — use their byte-level BPE (tiktoken) instead of reconstructing it.
- You want a single fast library covering BPE / WordPiece / Unigram with a modern training API — huggingface/tokenizers is the more active choice for new projects.
- You need to extend or mutate a vocabulary online — SentencePiece's fixed-vocab contract is the wrong tool.

## Alternatives

- huggingface/tokenizers — Rust, fast, trains BPE/WordPiece/Unigram; use when you want the mainstream modern training and serving library rather than a `.model` protobuf.
- openai/tiktoken — byte-level BPE, encode-only (no training); use when you need exact GPT-family tokenization at high speed.
- rsennrich/subword-nmt — the original reference BPE implementation; use for reproducing pre-2018 NMT results.
- glample/fastBPE — minimal C++ BPE; use when you want just BPE with no normalization or unigram machinery.
- huggingface/transformers — wraps SentencePiece for many models; use when you want the tokenizer coupled to a model's `from_pretrained` rather than managed standalone.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-03 | Repository created; C++ core + CLI tools[^4]. |
| paper | 2018-11 | EMNLP 2018 system demonstration paper published[^1]. |
| unigram | 2018 | Unigram LM segmentation + subword regularization introduced[^2]. |
| 0.1.9x | 2020–2023 | Long 0.1.x series; byte_fallback, Python wheel improvements. |
| 0.2.0 | 2024 | 0.2.x line; continued wheel/build modernization (SLSA 3 provenance). |

Exact per-tag release dates in the 0.1.x/0.2.x series are not restated here to avoid error; see the repository releases page for the authoritative list.

## References

[^1]: Kudo, T. & Richardson, J. "SentencePiece: A simple and language independent subword tokenizer and detokenizer for Neural Text Processing." EMNLP 2018 (system demonstrations). https://aclanthology.org/D18-2012/
[^2]: Kudo, T. "Subword Regularization: Improving Neural Network Translation Models with Multiple Subword Candidates." ACL 2018. https://arxiv.org/abs/1804.10959
[^3]: Sennrich, R., Haddow, B. & Birch, A. "Neural Machine Translation of Rare Words with Subword Units." ACL 2016. https://aclanthology.org/P16-1162/
[^4]: SentencePiece repository. https://github.com/google/sentencepiece

## Tags

tokenizer, subword, bpe, unigram-lm, nlp, text-processing, cpp, python-bindings, machine-translation, llm-preprocessing, multilingual, apache-2.0
