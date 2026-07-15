# sloria/TextBlob

> A thin, Pythonic convenience layer over NLTK and pattern for common NLP tasks — POS tagging, noun-phrase extraction, and lexicon sentiment in a few lines.

[GitHub repo](https://github.com/sloria/TextBlob) ·
[Official website](https://textblob.readthedocs.io/) ·
[License: MIT](https://github.com/sloria/TextBlob/blob/dev/LICENSE)

## Overview

TextBlob is a text-processing library created by Steven Loria, first released in 2013[^1]. Its stated goal is ergonomics: wrap the two established but verbose NLP toolkits — NLTK and pattern — behind a single `TextBlob` object so that tagging, tokenization, sentiment, and noun-phrase extraction read like attribute access rather than pipeline assembly. The README puts it plainly: it "stands on the giant shoulders of NLTK and pattern, and plays nicely with both."[^2]

The library is best understood as a teaching and prototyping tool. It is a fixture of introductory NLP courses, tutorials, and quick scripts precisely because `TextBlob(text).sentiment.polarity` requires no model download decisions, no pipeline configuration, and no ML background. The defining tradeoff is that this simplicity is bought with dated, lexicon-based methods: the default sentiment analyzer is pattern's hand-built adjective lexicon, and the tagger and parser are classical (rule/statistical) rather than neural. Accuracy on modern text — social media, domain jargon, sarcasm — is mediocre by 2020s standards.

TextBlob went through a long quiet period and has since had a maintenance revival: packaging was modernized, Python 2 and old Python 3 versions were dropped, and the Google-Translate-backed features were removed after the underlying endpoint stopped being usable[^3]. It remains genuinely maintained for compatibility, not actively developed toward new NLP capabilities.

## Getting Started

```bash
pip install -U textblob
python -m textblob.download_corpora   # pulls the NLTK data TextBlob needs
```

```python
from textblob import TextBlob

blob = TextBlob("TextBlob is amazingly simple to use. What great fun!")

blob.tags            # [('TextBlob', 'NN'), ('is', 'VBZ'), ('amazingly', 'RB'), ...]
blob.noun_phrases    # WordList(['textblob'])
blob.sentiment       # Sentiment(polarity=0.39, subjectivity=0.62)

for sentence in blob.sentences:
    print(sentence.sentiment.polarity)   # per-sentence scores

blob.words[0].pluralize()          # 'TextBlobs'
TextBlob("I havv goood speling.").correct()   # 'I have good spelling.'
```

The `download_corpora` step is not optional for most features — POS tagging, tokenization, and WordNet access all read NLTK data files at runtime, and a fresh environment will raise `LookupError` from deep inside NLTK if they are missing.

## Architecture / How It Works

TextBlob is a facade, not an engine. Almost every method delegates to NLTK or to a vendored copy of pattern's code, and the design is deliberately pluggable through small strategy classes:

- **Sentiment** — two analyzers. The default `PatternAnalyzer` scores text against pattern's English adjective lexicon and returns a `(polarity, subjectivity)` pair; polarity is a lexicon average, not a model prediction, so it is fast, deterministic, and blind to context and negation beyond simple heuristics. The alternative `NaiveBayesAnalyzer` is trained on an NLTK movie-review corpus and returns `(classification, p_pos, p_neg)`; it is slower and domain-biased toward reviews.
- **POS tagging** — `PatternTagger` (from pattern) or `NLTKTagger`. You select by passing `pos_tagger=` to the constructor.
- **Noun-phrase extraction** — `FastNPExtractor` by default, or `ConllExtractor` (chunker trained on the CoNLL-2000 corpus) for better phrase boundaries at higher cost.
- **Tokenization** — pluggable `BaseTokenizer` subclasses wrapping NLTK's word and sentence tokenizers.
- **Spelling correction** — a direct implementation of Peter Norvig's edit-distance-plus-word-frequency approach, driven by a bundled frequency file. It is per-word and context-free.
- **Classification** — `NaiveBayesClassifier` and `DecisionTreeClassifier` thinly wrap NLTK's classifiers, adding TextBlob's feature-extraction convenience over `(text, label)` tuples.

The `TextBlob`, `Sentence`, and `Word` types share a `BaseBlob` mixin so that a blob, a sentence, and a word expose the same interface where it makes sense. WordNet access on `Word` is a straight pass-through to NLTK's `wordnet` corpus reader. Because the analyzers, taggers, and extractors are constructor-injected, swapping in a custom implementation is the intended extension path — but the abstractions are shallow, and anything beyond the built-ins means reaching back down into NLTK directly.

## Production Notes

- **This is a prototyping library, not a production NLP stack.** Its sentiment and tagging quality lags spaCy and transformer models by a wide margin. Teams routinely start with TextBlob for a demo and then discover the polarity scores are too coarse for real decisions.
- **Corpus data is a deployment footgun.** `textblob.download_corpora` writes NLTK data to a user/home directory that may not exist or be writable in containers, serverless functions, or read-only images. The fix is to run the download during image build and/or set `NLTK_DATA` to a baked-in path. A missing corpus surfaces as a runtime `LookupError` on first use, not at import.
- **Translation and language detection are gone.** Earlier versions offered `TextBlob.translate()` and `.detect_language()` backed by an undocumented Google Translate endpoint. Google's changes broke it, and the feature was removed[^3]. Code and tutorials that still call these methods will fail — do not adopt TextBlob for translation.
- **Sentiment is a lexicon average, and it shows.** `PatternAnalyzer` has no real handling of negation scope, intensifiers beyond a small table, sarcasm, or domain vocabulary. Scores cluster near zero on neutral factual text and are easily fooled. For social-media tone specifically, cjhutto/vaderSentiment is a better rule-based choice; for accuracy, a fine-tuned transformer.
- **Performance is adequate but single-threaded and CPU-bound.** There is no batching API; processing large corpora means your own loop plus multiprocessing. First use of the Naive Bayes analyzer or the CoNLL extractor pays a corpus-load cost.
- **Determinism cuts both ways.** The lexicon path gives stable, reproducible scores across runs and versions — useful for tests — but that stability is also why the numbers never improve.

## When to Use / When Not

**Use when:**
- You want the shortest path from a string to POS tags, tokens, noun phrases, or a rough sentiment score.
- You're teaching, prototyping, or scripting, and readability matters more than accuracy.
- You need a light dependency for classical NLP without pulling in large model files.
- You already rely on NLTK and want a friendlier surface over it.

**Avoid when:**
- You need production-grade accuracy on sentiment, NER, or tagging — use spaCy or transformers.
- Your text is social media, multilingual, or domain-specific jargon.
- You need translation or language detection (removed).
- You need GPU acceleration, batching, or modern embeddings — TextBlob offers none.

## Alternatives

- nltk/nltk — the toolkit TextBlob wraps; use it directly when you need control over tokenizers, corpora, or classifiers rather than convenience defaults.
- explosion/spaCy — use when you need fast, accurate production tagging, parsing, and NER with a real pipeline API.
- huggingface/transformers — use when sentiment/classification accuracy is the priority and you can run (or fine-tune) a model.
- cjhutto/vaderSentiment — use instead of TextBlob's default when you specifically want rule-based sentiment tuned for social-media text.
- clips/pattern — TextBlob's other parent library (web mining plus NLP); largely unmaintained, but the source of the default sentiment lexicon.
- flairNLP/flair — use when you want strong sequence labeling (NER/POS) via contextual embeddings with a still-approachable API.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013 | Initial release by Steven Loria; facade over NLTK + pattern[^1]. |
| 0.15.3 | 2018-11 | Long-lived stable release across the quiet period. |
| 0.16.0 | 2021 | Dropped Python 2; translation/detection deprecation begins[^3]. |
| 0.17.1 | 2021 | Maintenance release. |
| 0.18.0 | 2024 | Modernized packaging; removed Google-Translate-backed translation and language detection; dropped old Python versions[^3]. |

## References

[^1]: TextBlob project history and authorship — Steven Loria. https://github.com/sloria/TextBlob
[^2]: TextBlob README — "stands on the giant shoulders of NLTK and pattern." https://github.com/sloria/TextBlob/blob/dev/README.rst
[^3]: TextBlob changelog (translation/detection removal, packaging and Python-version changes). https://textblob.readthedocs.io/en/latest/changelog.html

## Tags

python, nlp, natural-language-processing, sentiment-analysis, pos-tagging, text-processing, nltk, pattern, prototyping, library
