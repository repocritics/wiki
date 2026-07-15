# huggingface/optimum

> Hugging Face's umbrella for hardware-specific model export, quantization, and accelerated inference/training — a family of vendor extensions behind one `optimum` namespace.

[GitHub repo](https://github.com/huggingface/optimum) ·
[Official website](https://huggingface.co/docs/optimum/index) ·
[License: Apache-2.0](https://github.com/huggingface/optimum/blob/main/LICENSE)

## Overview

Optimum is the layer between Hugging Face's model libraries (Transformers, Diffusers, TIMM, Sentence Transformers) and non-CUDA-eager execution targets: ONNX/ONNX Runtime, Intel OpenVINO, AWS Trainium/Inferentia, Intel Gaudi (HPU), NVIDIA TensorRT-LLM, ExecuTorch, AMD, and others[^1]. Its promise is that a model you already run with `AutoModelForSequenceClassification` can be exported and served on a target accelerator by swapping one class, keeping the tokenizer/pipeline code unchanged.

The defining fact about Optimum is that it is not one library. The repository `huggingface/optimum` is a relatively thin core plus a documentation and CLI hub; the substance lives in separately-versioned sibling packages — `optimum-onnx`, `optimum-intel`, `optimum-neuron`, `optimum-habana`, `optimum-quanto`, `optimum-executorch`, `optimum-amd`, `optimum-nvidia`[^2]. You install `optimum[onnxruntime]`, `optimum[openvino]`, `optimum[habana]`, etc., and pip pulls the matching extension. As of 2026 the ONNX and ONNX Runtime integration — historically the flagship use case — was moved out of the core repo into `optimum-onnx`, a structural split the README now flags prominently[^3].

The tension this creates: Optimum reads as a single coherent product in the docs, but in practice it is a coordination problem across a dozen repos, each tracking its own vendor SDK and its own compatible range of Transformers versions. The Transformers-parity API is genuine and valuable; the packaging and version-skew reality is the recurring source of pain.

## Getting Started

```bash
# Core package
python -m pip install optimum

# Accelerator-specific extras (eager upgrade avoids stale transitive deps)
pip install --upgrade --upgrade-strategy eager optimum[onnxruntime]
pip install --upgrade --upgrade-strategy eager optimum[openvino]
```

```python
# Export a Transformers model to ONNX and run it via ONNX Runtime.
# ORTModelForXXX mirrors AutoModelForXXX — same call sites, ORT backend.
from optimum.onnxruntime import ORTModelForSequenceClassification
from transformers import AutoTokenizer, pipeline

model_id = "distilbert-base-uncased-finetuned-sst-2-english"

# export=True converts the PyTorch checkpoint to ONNX on first load
model = ORTModelForSequenceClassification.from_pretrained(model_id, export=True)
tokenizer = AutoTokenizer.from_pretrained(model_id)

clf = pipeline("text-classification", model=model, tokenizer=tokenizer)
print(clf("Optimum swaps the backend, not the API."))
```

```bash
# CLI export path — no Python glue needed
optimum-cli export onnx --model distilbert-base-uncased distilbert_onnx/
```

Note: with the ONNX split, `from optimum.onnxruntime import ...` and the `optimum-cli export onnx` command are provided by `optimum-onnx`; installing the `optimum[onnxruntime]` extra pulls it in[^3].

## Architecture / How It Works

Optimum's core abstractions are consistent across backends:

- **Exporters** — config-driven graph export. Each supported architecture has an export config declaring input/output names, dynamic axes, and the ONNX opset (or the target IR). `optimum.exporters.onnx` traces the PyTorch model and emits a portable graph; the same pattern generalizes to TFLite and other targets[^4]. Model coverage is per-architecture and hand-maintained, so a brand-new architecture in Transformers is not automatically exportable.
- **`ORTModelForXXX` / `OVModelForXXX` runtime wrappers** — classes that load an exported graph and expose the same `from_pretrained` / `forward` / pipeline surface as the Transformers `AutoModel` family. This is what makes the swap "one line": the wrapper adapts the accelerator runtime's session API to the Transformers calling convention.
- **Optimization & quantization configs** — `ORTOptimizer`/`ORTQuantizer` (graph fusion, ORT-level optimizations, static/dynamic INT8) and, for Intel, NNCF via `optimum-intel`. `optimum-quanto` is a separate PyTorch-native quantization backend usable independently of any export step.
- **Trainer wrappers** — for training targets (Gaudi via `optimum-habana`'s `GaudiTrainer`, Trainium via `optimum-neuron`, ORT training), Optimum subclasses the Transformers `Trainer` so existing training scripts move over with minimal edits[^1].

Coupling story: each extension is tied to a vendor SDK (ONNX Runtime, OpenVINO/NNCF, Habana SynapseAI, AWS Neuron, TensorRT-LLM) and to a compatible window of Transformers. The core `optimum` package coordinates the CLI and shared export plumbing; the extensions carry the heavy, fast-moving vendor code. `BetterTransformer`, an earlier attention-acceleration path, was deprecated as PyTorch's native scaled-dot-product-attention (SDPA) absorbed the same optimizations — a representative case of Optimum features being upstreamed and retired.

## Production Notes

**The install matrix is the main footgun.** The `--upgrade --upgrade-strategy eager` incantation in the docs is not cosmetic: without it, pip will happily leave a stale `transformers`, `onnxruntime`, or extension pinned, producing import errors or silent wrong behavior. Pin the whole set (optimum, the extension, transformers, and the vendor runtime) together and upgrade them as a unit.

**Version skew across sibling repos is real.** `optimum`, `optimum-onnx`, `optimum-intel`, etc. release on independent cadences and each declares its own supported Transformers range. A Transformers upgrade for an unrelated reason can break an Optimum export path until the corresponding extension catches up. Treat the extension's changelog — not the core repo's — as the source of truth for your backend.

**Export is where models break, not inference.** Unsupported architecture, unsupported task, a too-new opset, or dynamic-shape handling are the common export failures. Optimum can only export architectures for which an export config exists; newly-released models often lag. When export succeeds but outputs diverge from PyTorch, suspect opset/quantization numerics before suspecting the runtime.

**Quantization is not free accuracy.** Dynamic INT8 is low-effort but can move logits enough to matter for ranking/threshold-sensitive tasks; static INT8 needs a representative calibration set. Validate task metrics on the quantized graph, not just that it loads and runs.

**The ONNX relocation is a live migration cost.** Code and tutorials written before the `optimum-onnx` split assumed ONNX lived in the core package. Imports still resolve through the `optimum.onnxruntime` path when the extra is installed, but pinned-version setups and vendored docs referencing old install commands need updating[^3].

**Per-backend operational character differs.** ONNX Runtime CPU/GPU is the broadly-portable default; OpenVINO targets Intel CPU/GPU/NPU; Gaudi, Trainium/Inferentia, and TensorRT-LLM are cloud/vendor-specific with their own driver and container requirements (TensorRT-LLM ships primarily as a Docker image, not a plain pip install)[^1]. There is no single "just deploy Optimum" story — the deployment shape is dictated by the accelerator.

## When to Use / When Not

**Use when:**
- You already use Transformers/Diffusers and want ONNX or OpenVINO inference without rewriting call sites.
- You target a specific accelerator (Gaudi, Inferentia/Trainium, Intel NPU) that Hugging Face maintains an extension for.
- You want CLI-driven, reproducible model export as a build step.
- You want quantization/graph optimization wired to the HF model hub and pipeline API.

**Avoid when:**
- You need bleeding-edge or exotic architectures immediately — export config coverage lags new models.
- You want a single dependency: the real work is spread across separately-versioned extensions, and keeping them coherent is ongoing.
- Your workload is high-throughput LLM serving — a dedicated inference server (vLLM, TensorRT-LLM directly) usually beats an exported graph behind the Transformers API.
- You are comfortable calling ONNX Runtime / OpenVINO directly and don't need the Transformers-parity wrappers.

## Alternatives

- microsoft/onnxruntime — the underlying inference engine; use it directly when you don't need the Transformers-shaped wrappers or the HF export configs.
- pytorch/executorch — on-device/edge PyTorch inference; use when targeting mobile/embedded (Optimum wraps it via optimum-executorch, but ExecuTorch is the substance).
- vllm-project/vllm — use for high-throughput, low-latency LLM serving where paged-attention and continuous batching matter more than API parity.
- huggingface/transformers — use directly when eager PyTorch on CUDA is fast enough and you don't need export/quantization at all.
- openvinotoolkit/openvino — use directly for Intel-only deployments when you want full control over the OpenVINO/NNCF toolchain without the HF layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-07 | Repository created; ONNX Runtime export/inference as the first backend[^5]. |
| 1.0 | 2022 | Stabilized `ORTModelForXXX` API and Transformers-parity pattern. |
| — | 2022–2024 | Extension packages spun out: optimum-intel, optimum-habana, optimum-neuron, optimum-quanto, optimum-executorch, etc.[^2] |
| — | 2024–2025 | `BetterTransformer` deprecated as PyTorch SDPA upstreamed equivalent gains. |
| — | 2025–2026 | ONNX / ONNX Runtime integration relocated to standalone `optimum-onnx`[^3]. |

Exact release dates for the sibling packages vary per repo; treat each extension's own changelog as authoritative.

## References

[^1]: Optimum documentation — index and hardware support overview. https://huggingface.co/docs/optimum/index
[^2]: Optimum extension repositories (optimum-onnx, optimum-intel, optimum-neuron, optimum-habana, optimum-quanto, optimum-executorch). https://github.com/huggingface
[^3]: Optimum README — "ONNX integration was moved to `optimum-onnx`" notice. https://github.com/huggingface/optimum
[^4]: Optimum exporters documentation. https://huggingface.co/docs/optimum/exporters/overview
[^5]: Repository metadata (created 2021-07-20), GitHub API. https://github.com/huggingface/optimum

## Tags

python, machine-learning, inference, quantization, onnx, onnxruntime, openvino, model-optimization, transformers, hardware-acceleration, huggingface, edge-inference
