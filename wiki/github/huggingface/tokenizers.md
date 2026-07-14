# huggingface/tokenizers

> A Rust implementation of the subword tokenizers behind modern LLMs, exposed through thin Python/Node bindings.

[GitHub repo](https://github.com/huggingface/tokenizers) ·
[Documentation](https://huggingface.co/docs/tokenizers) ·
[License: Apache-2.0](https://github.com/huggingface/tokenizers/blob/main/LICENSE)

## Overview

`tokenizers` is the library that turns text into the integer IDs a transformer
model actually consumes, and back again. It provides training and inference for
the three subword algorithms that dominate the field — Byte-Pair Encoding (BPE),
WordPiece, and Unigram — plus the full pre-processing pipeline around them:
normalization, pre-tokenization, post-processing (special tokens), truncation,
and padding. The core is written in Rust; the artifact most people install is
the Python binding (`pip install tokenizers`), built with PyO3/maturin[^1].

Its reason to exist is speed. Reference tokenizer implementations in pure Python
were a real bottleneck for large-corpus training and high-throughput inference.
The Rust core tokenizes with parallelism (via `rayon`) and, per the project's own
README, processes on the order of a gigabyte of text in under 20 seconds on a
server CPU[^2]. The defining tension is that this is deliberately a *low-level*
library: it gives you a `Tokenizer` object and expects you to assemble or load
the right normalizer/pre-tokenizer/model/post-processor. Most users never touch
it directly — they hit it through `transformers`' `AutoTokenizer`, which wraps
`tokenizers` as its "fast" tokenizer backend[^3].

The second thing to understand is the `tokenizer.json` file: `tokenizers`
serializes an entire configured pipeline — vocab, merges, normalizer rules,
added tokens — into one JSON file. That file is the portable, framework-agnostic
unit that ships alongside model weights on the Hub and lets any binding load an
identical tokenizer.

## Getting Started

```bash
pip install tokenizers
# or from source, Python binding subdirectory:
# pip install "git+https://github.com/huggingface/tokenizers.git#subdirectory=bindings/python"
```

```python
from tokenizers import Tokenizer
from tokenizers.models import BPE
from tokenizers.pre_tokenizers import Whitespace
from tokenizers.trainers import BpeTrainer

# Build a BPE tokenizer from scratch and train a vocabulary.
tokenizer = Tokenizer(BPE(unk_token="[UNK]"))
tokenizer.pre_tokenizer = Whitespace()

trainer = BpeTrainer(special_tokens=["[UNK]", "[CLS]", "[SEP]", "[PAD]", "[MASK]"])
tokenizer.train(files=["wiki.train.raw", "wiki.valid.raw"], trainer=trainer)

output = tokenizer.encode("Hello, y'all! How are you 😁 ?")
print(output.tokens)   # ['Hello', ',', 'y', "'", 'all', '!', 'How', 'are', 'you', '[UNK]', '?']
print(output.ids)      # aligned integer ids

tokenizer.save("tokenizer.json")            # one-file, portable
reloaded = Tokenizer.from_file("tokenizer.json")
```

To use a pretrained tokenizer instead of training, `Tokenizer.from_pretrained("bert-base-uncased")`
pulls the `tokenizer.json` from the Hub.

## Architecture / How It Works

A `Tokenizer` is a pipeline of four swappable stages, applied in order:

1. **Normalizer** — Unicode-level cleanup: NFC/NFD/NFKC, lowercasing, accent
   stripping, custom replacements. Composable via `Sequence`.
2. **Pre-tokenizer** — splits the normalized string into word-like chunks
   (whitespace, punctuation, byte-level, Metaspace for SentencePiece-style).
   This is where the "words" that the model algorithm then subdivides come from.
3. **Model** — the actual subword algorithm: `BPE`, `WordPiece`, or `Unigram`.
   This owns the vocab and the merge/scoring logic.
4. **Post-processor** — inserts special tokens (`[CLS]`, `[SEP]`, BERT-style
   pairs), builds `type_ids` and the attention-relevant offsets.

The output of `encode` is an `Encoding` carrying `ids`, `tokens`, `attention_mask`,
`type_ids`, and — importantly — **offsets**: byte spans mapping each token back to
its exact slice of the original input. Alignment tracking survives normalization,
so you can always recover which characters produced a token. This is what makes
`tokenizers` usable for span-labeling tasks (NER, extractive QA) where losing the
character mapping would be fatal.

The Rust crate (`tokenizers/`) is the source of truth. Bindings for Python and
Node.js live under `bindings/` and are thin FFI wrappers; a third-party Ruby
binding is maintained externally[^2]. Training and batch encoding release the GIL
and parallelize across CPU cores with `rayon`, which is where the throughput
advantage over pure-Python tokenizers comes from.

## Production Notes

- **You almost always want the offsets, and they cost nothing to keep.** Code
  that reconstructs character spans by re-searching the input string is a common
  and avoidable bug — use `Encoding.offsets`.
- **Determinism depends on the whole pipeline, not just the vocab.** Two
  tokenizers with identical vocabularies can produce different token streams if
  their normalizers or pre-tokenizers differ. Ship and pin the full
  `tokenizer.json`, not just `vocab.txt`/`merges.txt`. The legacy split-file
  format loses pipeline configuration.
- **Fast vs. slow mismatch.** `transformers` has both a "slow" (pure-Python)
  and "fast" (this library) tokenizer per model. They are usually but not always
  identical; edge cases in normalization or added-token handling have
  historically diverged. If you trained/evaluated with one, serve with the same.
- **Added tokens are order- and flag-sensitive.** `add_tokens` vs.
  `add_special_tokens`, and the `special`/`normalized` flags, change how strings
  are matched and whether they survive normalization. Adding tokens after the
  fact does not retrain merges — a multi-piece word won't collapse into one id.
- **Parallelism warning you will see.** When a `tokenizers`-backed process is
  forked (common with PyTorch `DataLoader` workers), the library prints a
  deadlock warning and disables its own parallelism unless you set
  `TOKENIZERS_PARALLELISM=false` (or `true`) explicitly. Set it in data-loading
  pipelines to silence the noise and make behavior deterministic.
- **Byte-level BPE has no `[UNK]`.** GPT-style byte-level tokenizers can encode
  any input, so an unknown-token fallback never fires — different failure mode
  from WordPiece, where out-of-vocab genuinely maps to `[UNK]`.
- **Not a drop-in for OpenAI's `tiktoken`.** Both do BPE, but vocabularies,
  special-token conventions, and byte handling differ. Token *counts* will not
  match between the two for the same string.

## When to Use / When Not

**Use when:**
- You need to train a new subword vocabulary on a domain corpus, fast.
- You want a portable, one-file tokenizer that loads identically across Rust,
  Python, and Node.
- You need character-accurate offset alignment for span tasks.
- You're building the tokenizer layer under a custom (non-`transformers`) stack.

**Avoid / reach for something else when:**
- You just want to tokenize for an existing model — use `transformers`'
  `AutoTokenizer`, which wraps this and handles model-specific config for you.
- You only need to *count* tokens for an OpenAI model — use `tiktoken`, whose
  vocabularies match those APIs.
- You want SentencePiece's exact training semantics/`.model` format — use the
  upstream `sentencepiece` (though `tokenizers` can load and run Unigram models).

## Alternatives

- google/sentencepiece — the original C++ Unigram/BPE trainer; use when you need
  its exact `.model` format or its language-agnostic training semantics.
- openai/tiktoken — fast BPE for OpenAI models; use when you need token counts
  that match GPT APIs, not general-purpose training.
- huggingface/transformers — the high-level layer that wraps this library; use
  when you want per-model tokenizer config handled for you rather than assembling
  a pipeline by hand.
- rsennrich/subword-nmt — the classic reference BPE implementation; use only for
  reproducing older NMT papers, not for production.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019-11 | Repo created; Rust core + Python binding[^4]. |
| 0.8 | 2020 | Pipeline refactor: normalizer/pre-tokenizer/model/post-processor stages. |
| 0.10–0.13 | 2021–2022 | `tokenizer.json` single-file format matured; broad `transformers` fast-tokenizer coverage. |
| 0.19–0.20 | 2024 | PyO3/maturin build modernization, wider prebuilt wheels. |
| 0.21.x | 2025–2026 | Current line; ongoing perf and binding maintenance[^2]. |

## References

[^1]: `tokenizers` PyPI project (Python binding). https://pypi.org/project/tokenizers/
[^2]: Project README — features, performance claim, and language bindings. https://github.com/huggingface/tokenizers
[^3]: Transformers docs, "Fast tokenizers" — `AutoTokenizer` backed by this library. https://huggingface.co/docs/transformers/main_classes/tokenizer
[^4]: GitHub repository metadata (created 2019-11-01). https://github.com/huggingface/tokenizers

## Tags

rust, python, nlp, tokenizer, bpe, wordpiece, subword, huggingface, transformers, machine-learning, text-preprocessing
