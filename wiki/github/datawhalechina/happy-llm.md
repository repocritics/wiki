# datawhalechina/happy-llm

A free, systematic Chinese-language tutorial that teaches large language models from NLP fundamentals through building and training your own model.

## What it is

Happy-LLM is an open educational project from Datawhale that walks readers from basic NLP concepts up to implementing and training a small LLM by hand. It is aimed at students, researchers, and LLM enthusiasts who want to understand the theory (Transformer, attention, pretraining) and the practice (building LLaMA2, pretraining, fine-tuning, RAG, and agents) rather than just calling an API. The material is delivered as a structured, chapter-by-chapter book with accompanying code.

## Key features

- A seven-chapter progression: NLP basics, the Transformer architecture, pretrained language models, large language models, hands-on model building, training practice, and applications.
- Hands-on implementation of a LLaMA2-style model in PyTorch, including training a tokenizer and pretraining a small LLM.
- A training-practice chapter covering pretraining, supervised fine-tuning, and LoRA/QLoRA parameter-efficient fine-tuning, with a work-in-progress section on preference alignment.
- An applications chapter covering model evaluation, retrieval-augmented generation (RAG), and a simple agent implementation, each with runnable code.
- Released trained checkpoints (a 215M base model and an SFT model) on ModelScope, plus a free PDF and companion slides.
- An "Extra Chapter" section of community-contributed blog posts accepted via pull request.

## Tech stack

- Content authored primarily as Markdown chapters and Jupyter notebooks (the repository's primary language).
- Python, with per-chapter `requirements.txt` files and a recommendation to use a separate environment per chapter.
- PyTorch for the from-scratch model implementation in chapter five.
- The Hugging Face Transformers training stack introduced in chapter six for higher-level training workflows.
- Standalone RAG and Agent example directories with their own dependencies and demos.

## When to reach for it

- Learning how a decoder-only LLM works by building one end to end rather than treating it as a black box.
- Following a structured curriculum that connects Transformer theory to concrete training code.
- Practicing the full training pipeline — pretraining, SFT, and LoRA/QLoRA fine-tuning — on a small, tractable model.
- Getting an introductory, code-backed look at RAG and agents in Chinese.

## When *not* to reach for it

- Production model training at scale; the worked example is a roughly 215M-parameter model meant for learning.
- Readers who need English-first material — the primary text is Chinese, though an English README and translated pages exist.
- Anyone wanting a plug-and-play library or framework; this is a teaching resource, not a package you depend on.
- Commercial reuse without checking the license, which is non-commercial (CC BY-NC-SA 4.0).

## Maturity signal

With more than 32,000 stars, pushes continuing into 2026, and a complete set of finished chapters, the project is a well-established and actively maintained learning resource rather than an experiment. Its status markers show the core curriculum complete and only supplementary material (preference alignment, extra blogs) still in progress. Backing by the Datawhale open-source community and named project leads adds to its credibility.

## Alternatives

- rasbt/LLMs-from-scratch — use instead when you want an English, book-length "build an LLM from scratch" course.
- karpathy/nanoGPT — use instead when you want a minimal, code-first GPT training example without a written curriculum.
- datawhalechina/self-llm — use instead when your goal is using and fine-tuning existing open models rather than understanding their internals; this project is presented as the follow-up to self-llm.
- Hugging Face LLM/NLP Course — use instead when you want to learn primarily through the Transformers ecosystem's own tooling.

## Notes

The published PDF deliberately includes a non-intrusive Datawhale watermark, which the authors added to discourage resellers from repackaging the free material. The project also explicitly splits dependencies per chapter to reduce version conflicts, an unusual amount of care for a tutorial repository.

## Tags

llm, machine-learning, deep-learning, nlp, transformer, tutorial, pytorch, rag, agent, fine-tuning, jupyter-notebook
