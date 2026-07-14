# gradio-app/gradio

> Build a web UI for any Python function in a few lines, and share it over a temporary public URL.

[GitHub repo](https://github.com/gradio-app/gradio) ·
[Official website](https://gradio.app) ·
[License: Apache-2.0](https://github.com/gradio-app/gradio/blob/main/LICENSE)

## Overview

Gradio is a Python library for wrapping a machine-learning model, an API, or any arbitrary function in a browser UI without writing HTML, CSS, or JavaScript. It began as a research tool (the ICML HILL 2019 paper by Abid et al.[^1]) and became the default way to demo models on the Hugging Face Hub after Hugging Face acquired the project in December 2021[^2]. Today its main constituencies are ML researchers publishing reproducible demos, teams building internal tools, and anyone who needs a quick front end for a Python backend.

The defining tension is *demo tool versus application framework*. The high-level `gr.Interface` produces a working UI from a function signature and a list of component names, which is unbeatable for a five-minute model demo. The low-level `gr.Blocks` API exposes layout, event wiring, and shared state, and people push it much further than "demo" — the Automatic1111 Stable Diffusion WebUI is a Gradio Blocks app. The library carries the weight of both audiences: convenience defaults tuned for throwaway demos, and an event/queue system that has to hold up under real traffic. Most Gradio pain comes from treating a demo-shaped default as if it were production-hardened.

Gradio 5 (October 2024) was a substantial modernization — server-side rendering, a refreshed component set, and a third-party security audit by Trail of Bits[^3] — but the programming model has been stable since the 4.x line.

## Getting Started

```bash
pip install --upgrade gradio   # requires Python 3.10+
```

```python
import gradio as gr

def greet(name, intensity):
    return "Hello, " + name + "!" * int(intensity)

demo = gr.Interface(
    fn=greet,
    inputs=["text", "slider"],
    outputs=["text"],
)

demo.launch()   # serves on http://localhost:7860
```

Run `python app.py`, or `gradio app.py` for hot-reload during development. Pass `demo.launch(share=True)` to expose a temporary public `*.gradio.live` URL. For layout control beyond input/output lists, drop to `gr.Blocks`:

```python
with gr.Blocks() as demo:
    inp = gr.Textbox(label="Name")
    out = gr.Textbox(label="Greeting")
    inp.change(fn=lambda x: f"Hello {x}", inputs=inp, outputs=out)
demo.launch()
```

## Architecture / How It Works

A Gradio app is a Python object graph that is served by an ASGI backend and rendered by a compiled frontend:

- **Backend** — a FastAPI/Starlette app running under uvicorn. Your `Interface`/`Blocks` object is serialized to a config JSON that the frontend consumes; component values round-trip through auto-generated HTTP endpoints (and the app is queryable programmatically via `gradio_client`).
- **Frontend** — a Svelte application (built with Vite, styled with Tailwind). Each component is a Svelte package published under `@gradio/*`. Gradio 5 added optional SSR, which renders the initial page in a Node.js process for faster first paint.
- **Event model** — UI interactions (`.click`, `.change`, `.submit`) map to listener functions on the server. Return values update the targeted output components.
- **Queue** — every event listener is backed by a queue that mediates concurrency. Long-running or GPU-bound functions are serialized through it so a burst of users does not oversubscribe the machine. Streaming outputs (generators, `yield`) and progress updates are delivered over Server-Sent Events; SSE replaced the earlier WebSocket transport in the 4.x rewrite.
- **State** — `gr.State` holds per-session values in server memory keyed by a session hash. This is in-process memory, which is the single most important fact for anyone scaling beyond one worker.
- **Custom components** — since 4.0, third parties can build and `pip`-publish their own components (`gradio cc`), each shipping a Python class plus a Svelte frontend bundle.

The share feature is not a local hack: `share=True` opens a tunnel through Gradio-operated FRP (fast reverse proxy) infrastructure, so traffic to the `gradio.live` URL is relayed to your machine. The compute still runs locally; only the ingress is hosted.

## Production Notes

- **Single process, in-memory state — scaling is the footgun.** The queue and `gr.State` live in the worker's memory. Running multiple uvicorn workers or replicas without sticky sessions breaks state and queue accounting, because a follow-up request can land on a worker that never saw the session. Scale vertically, or put a session-affinity load balancer in front and treat each replica as independent.
- **`share=True` is for demos, not deployment.** The link is temporary (it expires) and routes through third-party tunnel infrastructure. For anything durable, serve the app directly (`server_name="0.0.0.0"`) behind your own reverse proxy, or host on Hugging Face Spaces.
- **Mount into an existing server rather than fight it.** `gr.mount_gradio_app(fastapi_app, demo, path="/gradio")` embeds a Gradio app inside your own FastAPI service, which is cleaner than trying to reshape `launch()` for production ingress, TLS, and auth.
- **Security has real history.** Gradio has shipped CVEs around file access and path traversal from its endpoints (notably in the 3.x era); keep the version current, be deliberate about `allowed_paths`/`blocked_paths`, and do not expose an unauthenticated instance that can read arbitrary files. Gradio 5's Trail of Bits audit[^3] addressed a batch of these, but staying patched is on the operator.
- **Built-in auth is minimal.** `launch(auth=...)` gives HTTP basic-style login, not an identity system. Front real deployments with a proper auth proxy or Spaces' access controls.
- **Concurrency defaults changed across majors.** The 3.x → 4.x → 5.x line reworked queueing and concurrency semantics (per-event `concurrency_limit`, default queue-on). Pin your major version and re-read the queue docs when upgrading; throughput behavior does not silently carry over.
- **Version skew between client and server.** `gradio` and `gradio_client` (and hosted Spaces) are versioned together; an app built on one major and queried by a client on another can mismatch on the config/protocol. Keep them aligned.

## When to Use / When Not

**Use when:**
- You need a UI for an ML model or Python function fast, with no frontend work.
- You are publishing a reproducible research demo, especially on Hugging Face Spaces.
- You want a chatbot UI with streaming out of the box (`gr.ChatInterface`).
- You are building an internal tool where a Python-only stack is a feature, not a limitation.

**Avoid when:**
- You need a public-facing product with custom design, complex routing, and SEO — reach for a real web framework.
- You require horizontal scale-out with shared session state; Gradio's in-memory model fights this.
- Your app is dashboard-shaped (many charts, filters, cross-linked views) — a dashboard framework fits better.
- You want fine-grained control over markup and interaction that the component model does not expose.

## Alternatives

- streamlit/streamlit — script-rerun model for data apps and dashboards; use instead when you want a scripted, top-to-bottom data tool rather than a function-wrapping demo.
- plotly/dash — callback-based analytical dashboards on Flask/React; use when the app is chart- and filter-heavy and you need enterprise deployment options.
- holoviz/panel — dashboarding tuned for the scientific Python and notebook ecosystem; use when you live in Jupyter/HoloViews.
- chainlit/chainlit — conversational-AI and LLM app UIs specifically; use when your product is a chat/agent app rather than a general model demo.
- google/mesop — Python UI framework for internal web apps; use when you want more app-structure control while staying in Python.

## History

| Version | Date | Notes |
|---------|------|-------|
| paper | 2019 | ICML HILL paper introduces Gradio for sharing/testing ML models[^1]. |
| — | 2021-12 | Hugging Face acquires Gradio; becomes the default Spaces SDK[^2]. |
| 3.0 | 2022 | `gr.Blocks` maturity; broad component library. |
| 4.0 | 2023 | Custom components, SSE-based streaming replacing WebSockets. |
| 5.0 | 2024-10 | SSR, refreshed components, Trail of Bits security audit[^3]. |

## References

[^1]: Abid et al., "Gradio: Hassle-Free Sharing and Testing of ML Models in the Wild," ICML HILL 2019. https://arxiv.org/abs/1906.02569
[^2]: Hugging Face, "Gradio is joining Hugging Face!" — 2021-12. https://huggingface.co/blog/gradio-joins-hf
[^3]: Gradio blog, "Gradio 5 is here" (security audit by Trail of Bits) — 2024-10. https://huggingface.co/blog/gradio-5

## Tags

python, machine-learning, web-ui, ml-demos, fastapi, svelte, hugging-face, chatbot, data-science, low-code
