# jupyterlab/jupyterlab

> The web-based IDE for Project Jupyter — notebooks, terminals, and a file browser in one extensible Lumino-based frontend.

[GitHub repo](https://github.com/jupyterlab/jupyterlab) ·
[Official website](https://jupyterlab.readthedocs.io/) ·
[License: BSD-3-Clause](https://github.com/jupyterlab/jupyterlab/blob/main/LICENSE)

## Overview

JupyterLab is the next-generation frontend for Project Jupyter, succeeding the "classic" Jupyter Notebook interface[^1]. It is not a notebook format and not a kernel — it is a browser application (written in TypeScript) that talks to a separate Python backend (Jupyter Server) over REST and WebSocket, which in turn manages language kernels that actually execute code. Understanding that three-layer split — frontend, server, kernel — is the key to reasoning about everything else, including why so much "JupyterLab" behavior is really the server's or the kernel's[^2].

The defining tension is the extension system. JupyterLab is built as a set of composable plugins on top of the Lumino widget/DI framework (formerly PhosphorJS), and nearly every panel you see is a swappable extension. This makes it uniquely customizable, but historically it also made it fragile: extensions were tied to the exact JupyterLab major version, and a `jupyter lab build` step (requiring Node.js on the deployment machine) was needed to install them. JupyterLab 3 introduced *prebuilt* (federated) extensions distributed as wheels via PyPI/conda, removing the Node/build requirement for end users[^3]. This is the single most important thing to know when choosing how to deploy extensions.

JupyterLab is the reference frontend behind Notebook 7, JupyterLite, and most of the hosted-notebook products in the ecosystem, so its component libraries have reach far beyond the `jupyter lab` command itself.

## Getting Started

```bash
pip install jupyterlab        # or: conda install -c conda-forge jupyterlab
jupyter lab                   # opens http://localhost:8888/lab in the browser
```

```bash
# Install a prebuilt extension — no Node.js / rebuild needed since v3
pip install jupyterlab-git
jupyter labextension list      # verify it registered

# Headless / server deployment: bind all interfaces, keep the token
jupyter lab --ip=0.0.0.0 --no-browser --ServerApp.token="$JUPYTER_TOKEN"
```

The kernel is chosen from whatever is installed in the environment (`ipykernel` for Python, `IRkernel` for R, etc.). JupyterLab does not ship kernels; a fresh install can only run Python if `ipykernel` is present, which the `jupyterlab` wheel pulls in transitively.

## Architecture / How It Works

**Frontend (this repo).** A monorepo of ~1600 files across dozens of `@jupyterlab/*` npm packages. The application core is a plugin registry: each plugin declares the tokens it *provides* and *requires*, and the system wires them via dependency injection at startup. UI is composed from **Lumino** widgets and the Lumino command/keybinding/dock-panel system — the drag-dock layout and command palette are Lumino, not bespoke JupyterLab code. Text editing moved to **CodeMirror 6** in JupyterLab 4, a full rewrite of the editor layer that broke extensions depending on CodeMirror 5 internals[^4].

**Server.** The frontend has no execution ability of its own. It calls **Jupyter Server** (`jupyter_server`), which exposes the contents API (files), the kernels/sessions API, and terminals. JupyterLab 4 requires `jupyter_server` 2.x; the old `notebook`-package server is gone. Kernel messaging is ZeroMQ between server and kernel, surfaced to the browser as a WebSocket.

**Extension distribution.** Two kinds coexist: *source extensions* (installed from npm, require `jupyter lab build` and therefore Node.js) and *prebuilt/federated extensions* (built ahead of time using webpack Module Federation, shipped as Python packages, dropped into `share/jupyter/labextensions`). Prebuilt is the modern default; it decouples extension builds from the core application build[^3].

**Real-time collaboration** is not in core. It lives in the separate `jupyter-collaboration` extension (Yjs-based CRDT documents) and must be installed and enabled explicitly[^5].

## Production Notes

**Single-user by default; multi-user needs JupyterHub.** `jupyter lab` serves one user with one server process and full shell/kernel access to the host. It is not an auth boundary. Anyone who reaches the port with the token can execute arbitrary code as the server user. Multi-tenant deployments use JupyterHub (or a hosted product) to spawn one server per user; do not put a raw `jupyter lab` on a public IP.

**Token/XSRF footguns.** The default security model is a per-session token in the URL. Reverse proxies that strip query strings, or `--ServerApp.token=""` "for convenience," routinely turn into open RCE. Terminate TLS in front and keep the token or a password.

**Extension version pinning.** Prebuilt extensions declare a compatible JupyterLab major range. Upgrading JupyterLab (e.g. 3 → 4) commonly disables extensions until each is updated by its author; there is no shim layer across majors. Pin `jupyterlab` in production environments and upgrade extensions deliberately, not with a blanket `pip install -U`.

**Large-notebook performance.** Notebook JSON embeds all outputs inline, so a notebook with megabytes of rendered images or DataFrame HTML becomes slow to load, diff, and version-control. JupyterLab 4 added **windowed (virtualized) rendering** of notebook cells, which greatly helps large notebooks but can break extensions that assumed every cell is present in the DOM[^4]. Strip outputs (`nbstripout`) before committing.

**Kernel state is invisible.** Out-of-order cell execution means the on-screen notebook can diverge from actual kernel state; "Restart & Run All" is the only reliable reproducibility check. Kernels also hold memory until explicitly shut down — idle kernels are a common cause of server OOM on shared hosts (`--MappingKernelManager.cull_idle_timeout`).

**Upgrade history worth knowing:** JupyterLab 3 reached end of maintenance on 2024-05-15 (critical fixes backported to 2024-12-31)[^1]; running v3 today is unsupported. The 3 → 4 jump (CodeMirror 6, Lumino 2, `jupyter_server` 2) is the most disruptive migration for extension authors.

## When to Use / When Not

**Use when:**
- You want a full IDE-like surface (notebooks + editor + terminal + file browser) in the browser.
- You're building or hosting a custom data-science environment and need the extension API.
- You're standing up multi-user notebooks via JupyterHub, or targeting JupyterLite/Notebook 7 whose components come from here.
- You need R, Julia, or other kernels alongside Python in one UI.

**Avoid when:**
- You want a single simple document view — Notebook 7 (`jupyter/notebook`) is lighter and document-centric.
- You want notebooks inside an existing editor — VS Code's Jupyter extension keeps you in one tool.
- You need reactive/reproducible-by-construction notebooks — marimo eliminates hidden-state bugs by design.
- You need zero-install, browser-only execution with no server — use JupyterLite.

## Alternatives

- jupyter/notebook — Notebook 7, rebuilt on JupyterLab components; lighter, single-document UX. Use when you want classic-notebook simplicity without the IDE surface.
- jupyterlite/jupyterlite — JupyterLab/Notebook running fully in-browser on WASM (Pyodide), no server. Use when you need static hosting and no backend.
- microsoft/vscode-jupyter — notebooks inside VS Code. Use when your team already lives in VS Code and wants Git/debugger integration there.
- marimo-team/marimo — reactive Python notebooks stored as `.py`, deterministic execution order. Use when hidden kernel state and non-reproducibility are the pain.
- quarto-dev/quarto-cli — publishing/rendering of notebooks to reports and sites. Use for the authoring-to-publication pipeline, not interactive compute.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x beta | 2018 | Public beta; classic Notebook still the default UI. |
| 1.0 | 2019-02 | First stable release; general availability[^1]. |
| 2.0 | 2020-02 | Editor/UX improvements, extension API maturation. |
| 3.0 | 2021-01 | Prebuilt (federated) extensions, visual debugger, multi-language UI[^3]. |
| 4.0 | 2023-06 | CodeMirror 6, Lumino 2, `jupyter_server` 2, windowed notebook rendering[^4]. |
| 3.x EOL | 2024-05-15 | End of maintenance for the 3.x line[^1]. |

## References

[^1]: JupyterLab README and "JupyterLab 3 end of maintenance," Jupyter Blog. https://blog.jupyter.org/jupyterlab-3-end-of-maintenance-879778927db2
[^2]: JupyterLab documentation — general architecture. https://jupyterlab.readthedocs.io/en/stable/
[^3]: Jupyter Blog, "JupyterLab 3.0 is released!" (prebuilt/federated extensions, debugger). https://blog.jupyter.org/jupyterlab-3-0-is-out-4f7635ccd6ba
[^4]: JupyterLab documentation, "Migrating from 3.x to 4.x" (CodeMirror 6, Lumino 2, windowed rendering). https://jupyterlab.readthedocs.io/en/stable/extension/extension_migration.html
[^5]: jupyter-collaboration extension (real-time collaboration, Yjs). https://github.com/jupyterlab/jupyter-collaboration

## Tags

typescript, jupyter, notebook, data-science, ide, interactive-computing, web-application, lumino, extensible, scientific-computing
