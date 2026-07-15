# apache/opennlp

> A JVM-native, classical-ML toolkit for the standard NLP pipeline — tokenize, tag, chunk, find names — with trainable models and no Python in sight.

[GitHub repo](https://github.com/apache/opennlp) ·
[Official website](https://opennlp.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/opennlp/blob/main/LICENSE)

## Overview

Apache OpenNLP is a machine-learning toolkit for natural language processing written entirely in Java. It covers the classic pre-transformer pipeline: sentence segmentation, tokenization, part-of-speech tagging, named-entity recognition, chunking, parsing, lemmatization, language detection, and coreference resolution. It began life around 2000 as a SourceForge project started by Jason Baldridge and Gann Bierner, entered the Apache Incubator in 2010, and graduated to an Apache top-level project in 2012[^1]. It is one of the oldest continuously maintained NLP libraries still in active development.

The defining characteristic is that OpenNLP is a *classical* statistical toolkit at heart. Its core learners are Maximum Entropy (MaxEnt), Perceptron, Naive Bayes, and — through an add-on module — libsvm-backed SVM[^2]. These are fast, small, CPU-only models you train yourself from annotated text; a trained model is a self-contained `.bin` file measured in megabytes, not gigabytes. Since 2.0 the project has bolted on a deep-learning path (`opennlp-dl`) that runs ONNX models — including BERT-style encoders — through the ONNX Runtime, but this is an adapter layer, not a reimplementation of the framework around neural nets[^3].

The tension for a 2026 adopter is exactly that heritage. OpenNLP will not beat a fine-tuned transformer on accuracy for entity recognition or classification. What it offers instead is a mature, dependency-light, Apache-2.0-licensed JVM library that runs a full NLP pipeline on commodity hardware with predictable latency, trains in minutes, and drops cleanly into Flink / NiFi / Spark streaming jobs[^2]. It is infrastructure for teams that live on the JVM and value determinism and licensing clarity over leaderboard scores.

## Getting Started

The 3.x line is modular; you import only the pieces you need. `opennlp-runtime` ships with the MaxEnt implementation; add other ML modules as required.

```xml
<dependency>
    <groupId>org.apache.opennlp</groupId>
    <artifactId>opennlp-runtime</artifactId>
    <version>${opennlp.version}</version>
</dependency>
<dependency>
    <groupId>org.apache.opennlp</groupId>
    <artifactId>opennlp-model-resolver</artifactId>
    <version>${opennlp.version}</version>
</dependency>
```

```java
// Tokenize with a trained model, then find person names.
try (InputStream tokIn = new FileInputStream("en-token.bin");
     InputStream nerIn = new FileInputStream("en-ner-person.bin")) {

    TokenizerME tokenizer = new TokenizerME(new TokenizerModel(tokIn));
    NameFinderME nameFinder = new NameFinderME(new TokenNameFinderModel(nerIn));

    String[] tokens = tokenizer.tokenize("Alan Turing was born in London.");
    Span[] names = nameFinder.find(tokens);           // spans over the token array

    for (Span s : names) {
        System.out.println(s.getType() + ": " +
            String.join(" ", Arrays.copyOfRange(tokens, s.getStart(), s.getEnd())));
    }
    nameFinder.clearAdaptiveData();  // reset per-document state between documents
}
```

The same operations are available from the `opennlp` CLI (`opennlp TokenizerME`, `opennlp TokenNameFinder`, etc.) for training, evaluation, and one-off runs without writing Java.

## Architecture / How It Works

Every component follows the same shape: a `*Model` class that wraps trained parameters loaded from a `.bin` file, and a `*ME` runner (Maximum Entropy being the historical default) that executes it. `TokenizerME`, `SentenceDetectorME`, `POSTaggerME`, `ChunkerME`, `LemmatizerME`, and `NameFinderME` all share this pattern. Models are produced by feature extraction over annotated training data feeding one of the pluggable learners.

The learners are the real substance. MaxEnt (trained with GIS, generalized iterative scaling) is the workhorse for sequence-labeling tasks; Perceptron and Naive Bayes are alternatives; SVM lives in a separate `opennlp-ml-libsvm` module backed by zlibsvm[^2]. Because these are log-linear / margin models over hand-defined feature functions, behavior is inspectable and reproducible in a way transformer black boxes are not — the same input and model always yield the same output.

The 3.x restructure split what used to be a monolithic `opennlp-tools` jar into focused modules: `opennlp-api` (interfaces), `opennlp-runtime` (core), the per-algorithm ML modules, `opennlp-formats` (training-data readers), `opennlp-dl` (ONNX adapter, with an `opennlp-dl-gpu` variant swapping in the GPU ONNX Runtime), plus extensions for Morfologik stemming, SymSpell spell-checking, and UIMA annotators[^2]. `opennlp-model-resolver` handles discovering and loading models from the classpath.

Named-entity recognition carries a stateful wrinkle: `NameFinderME` maintains *adaptive* feature data across a document to improve consistency, so you must call `clearAdaptiveData()` between documents or accuracy silently degrades. This is the single most common correctness bug in OpenNLP code.

## Production Notes

- **Bring your own models.** The pre-built demo models published on the Apache downloads site are for testing and getting started; the project explicitly tells you to train your own for real use[^4]. Domain-specific data almost always beats the generic demo models, which are dated and English-centric.
- **Thread safety changed in 3.0.** The core `*ME` classes are now thread-safe and a single instance can be shared across threads, eliminating the old per-thread pooling. The `ThreadSafe*ME` wrappers from 2.x still work but are deprecated[^2]. If you are on 2.x, treat `*ME` instances as *not* thread-safe.
- **Java version floor jumped hard.** 3.0.0 requires JDK 21; the 2.x line (maintained on the `opennlp-2.x` branch) requires JDK 17[^2]. There is no getting to 3.x without a modern JVM.
- **2.x → 3.x is low-risk but not zero.** The maintainers report no known breaking API changes — the split is structural — so you can keep depending on `opennlp-tools`, but the recommended move is to switch to `opennlp-runtime` plus the specific modules you use for a smaller footprint[^2].
- **A package rename is signposted.** The team plans to move the package namespace from `opennlp` to `org.apache.opennlp` in a future release (possibly 4.x)[^2]. Import statements across your codebase will change when that lands; budget for it.
- **Deep learning is opt-in and heavier.** `opennlp-dl` pulls in the ONNX Runtime native library and expects you to supply exported ONNX models; it is not a drop-in accuracy upgrade for the classical components, and the GPU variant adds CUDA-class deployment concerns.
- **Memory and startup are modest.** Loading a handful of `.bin` models costs tens to low-hundreds of MB and inference is CPU-bound and fast, which is why OpenNLP fits well inside streaming operators where a transformer would not.

## When to Use / When Not

**Use when:**
- You are on the JVM and want an NLP pipeline without a Python sidecar or model-server hop.
- You need permissive Apache-2.0 licensing (redistribution, embedding, closed-source products).
- Latency, determinism, and small model size matter more than last-percentage accuracy.
- You embed NLP inside stream processors (Flink, NiFi, Spark) or high-throughput services.
- You have annotated data and want to train task-specific models quickly on CPU.

**Avoid when:**
- You need state-of-the-art accuracy on NER / classification — fine-tuned transformers win.
- Your stack is Python-first; spaCy or Hugging Face are the natural fit.
- You want rich out-of-the-box multilingual neural models with no training effort.
- You need dependency parsing or semantics beyond OpenNLP's classical component set.

## Alternatives

- explosion/spaCy — reach for it when your stack is Python and you want a fast, modern, batteries-included pipeline with pretrained pipelines.
- stanfordnlp/CoreNLP — the other mature JVM toolkit; higher classical accuracy and richer annotations, but its GPL license is a blocker for many commercial products where OpenNLP's Apache-2.0 is not.
- stanfordnlp/stanza — use when you want neural, highly multilingual annotations from Python and can pay the model-download and latency cost.
- huggingface/transformers — use when accuracy is paramount and you can host transformer models (or call OpenNLP's own `opennlp-dl` ONNX path for a lighter JVM bridge).
- nltk/nltk — use for teaching, research, and prototyping in Python rather than production throughput.

## History

| Version | Date | Notes |
|---------|------|-------|
| SourceForge origins | ~2000 | Started by Jason Baldridge and Gann Bierner as a research project[^1]. |
| Apache Incubator | 2010 | Codebase donated; 1.5.x releases made under Apache[^1]. |
| Top-level project | 2012 | Graduated from the Incubator to an Apache TLP[^1]. |
| 1.9.x | 2019–2021 | Long-running stable line on the classical ML stack. |
| 2.0.0 | 2022 | Added the `opennlp-dl` ONNX Runtime adapter for neural model inference[^3]. |
| 3.0.0 | 2025 | Modular restructure, JDK 21 minimum, thread-safe `*ME` classes[^2]. |

## References

[^1]: Apache OpenNLP — project home and history. https://opennlp.apache.org/
[^2]: Apache OpenNLP README — modules, 2.x→3.x migration, thread safety, branch/Java policy. https://github.com/apache/opennlp
[^3]: Apache OpenNLP documentation — deep learning / ONNX (`opennlp-dl`). https://opennlp.apache.org/docs/
[^4]: Apache OpenNLP demo models (train your own for production use). https://downloads.apache.org/opennlp/models/

## Tags

java, nlp, machine-learning, named-entity-recognition, tokenization, part-of-speech-tagging, maximum-entropy, onnx, jvm, apache, text-processing
