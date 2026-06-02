# tensorflow/tensorflow

Google's open-source machine-learning framework — the original "industrial-scale TF" that dominated production ML for the second half of the 2010s and now coexists with PyTorch and JAX.

## What it is

A comprehensive machine-learning framework that spans research training (TensorFlow + Keras), production serving (TensorFlow Serving, TFX), edge deployment (TensorFlow Lite), and browser deployment (TensorFlow.js). The Python API is the day-to-day surface; the C++ core handles compilation and execution. TensorFlow 2's eager-mode + Keras-as-default-API was the major architectural rewrite that closed the usability gap with PyTorch.

## Key features

- Multi-target deployment: server (TF Serving), mobile/edge (TF Lite), browser (TF.js), microcontroller (TF Lite Micro).
- Keras as the default high-level API — model-building, training loops, and metrics share a Pythonic surface.
- XLA compilation for cross-device performance, particularly TPU.
- TFX (TensorFlow Extended) — production ML pipelines: data ingestion, validation, transform, model analysis, serving.
- TensorBoard for training visualization, baked into the ecosystem.
- DOI + Zenodo archiving for academic citation.

## Tech stack

- C++ core for graph compilation and execution.
- Python primary user API; bindings for C, Java, JavaScript, Swift (limited), Rust (community).
- XLA (Accelerated Linear Algebra) compiler for GPU/TPU code generation.
- Bazel-driven build system.
- Apache 2.0 licensed — the cleanest license posture among major ML frameworks.

## When to reach for it

- You're deploying ML to constrained edge devices (TF Lite is the canonical path).
- You need a TPU target — TF/JAX has first-class TPU support; PyTorch's TPU story remains second-class.
- You're maintaining an existing TF codebase or working in an organization that's standardized on TF / TFX.
- You need TensorBoard-grade training observability with minimal setup.

## When *not* to reach for it

- You're starting a new research project today — PyTorch and JAX have larger active research communities and faster iteration.
- You want maximum API stability — TF has been through major API rewrites (TF 1 → TF 2 → TF 2 + Keras 3).
- You need bleeding-edge model implementations — the open-source modeling community has consolidated around PyTorch and Hugging Face.

## Maturity signal

195k stars, 75k forks, Apache 2.0, last push the morning this page was generated — actively maintained. 10-year-old project with Google as primary sponsor. The CII Best Practices badge and DOI registration signal mature OSS hygiene. The 3,084 open-issues count is high in absolute terms but proportional to the surface area; bug-fix triage continues but feature velocity has slowed as the team prioritizes TPU/XLA and Keras 3.

## Alternatives

- `pytorch/pytorch` — use for research, modern model implementations, and most new training projects.
- `google/jax` — use for high-performance numerical computing with functional transformations.
- `tinygrad/tinygrad` — use when you want an educational, minimal-codebase ML framework.
- `pytorch/serve`, BentoML, Ray Serve — use when serving needs are the primary concern.

## Notes

The TF 2 / Keras 3 era unified the API across TF, JAX, and PyTorch backends — Keras can now run on PyTorch as a backend, blurring framework boundaries. Anyone choosing between TF and PyTorch today should weight team familiarity and deployment target heavily; both are production-viable, and the ecosystem-overlap is increasing. The Apache 2.0 license is permissive enough for almost any commercial reuse, including training of derivative models on TF source.

## Tags

machine-learning, deep-learning, neural-network, python, c-plus-plus, framework, distributed, gpu, tpu, tensorflow-lite, apache-license
