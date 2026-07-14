# streamlit/streamlit

> Turn a Python script into an interactive web app by rerunning the whole script on every interaction.

[GitHub repo](https://github.com/streamlit/streamlit) ·
[Official website](https://streamlit.io) ·
[License: Apache-2.0](https://github.com/streamlit/streamlit/blob/develop/LICENSE)

## Overview

Streamlit is a Python framework for building data and ML web apps without writing HTML, CSS, or JavaScript. It was open-sourced in October 2019 by Adrien Treuille, Amanda Kelly, and Thiago Teixeira, and the company was acquired by Snowflake in March 2022[^1]. The pitch is unchanged since launch: write a linear Python script, call functions like `st.slider` and `st.dataframe`, and get a live web app with widgets wired up for you. It is aimed at data scientists and ML engineers who want to ship an interactive prototype in an afternoon, not at web developers building a general-purpose product.

The defining design decision — and the source of nearly every Streamlit surprise — is the **rerun model**: on every widget interaction, Streamlit re-executes your entire script from top to bottom[^2]. There is no callback graph, no component lifecycle, no explicit state machine. This makes the beginner experience genuinely simple (the script reads like a notebook) but means that anything expensive, stateful, or order-dependent has to be deliberately guarded with caching and `st.session_state`. Streamlit is the most popular tool in its category by a wide margin, and its ecosystem (Community Cloud hosting, third-party components, chat/LLM primitives) reflects heavy adoption in the GenAI demo space.

## Getting Started

```bash
pip install streamlit
streamlit hello        # sanity-check the install
```

```python
# app.py
import streamlit as st

st.title("Square calculator")

x = st.slider("Pick a value", 0, 100, 10)   # returns current widget value
st.write(f"{x} squared is {x * x}")

# Widget values persist across reruns via keyed session state
if st.button("Reset note"):
    st.session_state.pop("note", None)
st.text_input("A note", key="note")
```

```bash
streamlit run app.py   # serves on http://localhost:8501
```

Every edit to `app.py` triggers a "Rerun / Always rerun" prompt in the browser, so the edit-refresh loop is fast.

## Architecture / How It Works

A running Streamlit app is two processes glued by a WebSocket. The **backend** is a Tornado server running your Python script; the **frontend** is a compiled React/TypeScript single-page app. On each interaction the frontend sends the widget state to the backend, the backend reruns the script top-to-bottom, and each `st.*` call emits a Protobuf "delta" message that the frontend applies to the DOM[^2]. The script is not a long-lived object — it is a function that runs to completion on every event.

Because the whole script reruns, two mechanisms exist to opt out of "recompute everything":

- **Caching.** `@st.cache_data` memoizes serializable return values (dataframes, API responses); `@st.cache_resource` memoizes singletons that should not be copied (DB connections, ML models). These replaced the older single `@st.cache` decorator, whose hashing heuristics were a long-standing source of confusion, in the 1.18 release[^3].
- **Session state.** `st.session_state` is a per-session dict that survives reruns, giving you somewhere to keep mutable state (counters, form drafts, chat history). Widgets bound with `key=` read and write it automatically[^4].

Later additions soften the rerun model without abandoning it: **`st.fragment`** marks a function whose interactions rerun only that fragment instead of the whole page, and **`st.navigation` / `st.Page`** provide programmatic multipage routing that superseded the earlier filesystem-based `pages/` convention[^5]. **Custom Components** embed an arbitrary React app in a sandboxed iframe with a two-way `Streamlit.setComponentValue` bridge, which is how the ecosystem adds things like `streamlit-aggrid` and drawable canvases[^6].

Everything runs server-side. The browser holds no application logic — it is a thin renderer — so all computation, memory, and session state live in the Python process.

## Production Notes

**Streamlit is stateful and server-bound, which fights horizontal scaling.** Each browser session pins to one server process over a persistent WebSocket, and all session state lives in that process's memory. You cannot round-robin requests across replicas behind a stateless load balancer; you need sticky sessions, and session state is lost if the process restarts or the connection drops. For multi-user production this is the first wall teams hit.

**The rerun model is a performance footgun.** Any code not wrapped in a cache or guarded by a state check runs on every keystroke-driven rerun. Loading a model or querying a warehouse at the top of the script — without `@st.cache_resource` — will re-run on every interaction and make the app crawl. Debugging "why is my app slow" almost always ends at "this expensive call wasn't cached."

**Concurrency is limited.** Streamlit historically ran script execution for a session on a single thread, and CPU-heavy work in one session can starve others on the same process. The common production pattern is one small app per container, an external cache/queue for heavy work, and horizontal scaling by running many single-app instances rather than one big multi-tenant server.

**Auth is not really built in.** For years there was no first-party authentication; teams put Streamlit behind an OAuth2 proxy, a reverse proxy, or Community Cloud's Google login. Native OIDC login (`st.login` / `st.logout`) arrived only in a 2025 release and covers authentication, not fine-grained authorization[^7]. Do not expose an internal-data app to the internet assuming Streamlit gates access.

**Upgrade friction is usually deprecations, not rewrites.** The API has been relatively stable since 1.0, but `st.cache` → `cache_data`/`cache_resource`, the `st.experimental_*` → stable renames, and the `pages/`-directory → `st.navigation` shift each required code changes. Pin the version; minor releases ship roughly monthly and occasionally change widget defaults or deprecate APIs.

**Customization has a ceiling.** Theming is a config block plus limited CSS injection (`st.markdown(..., unsafe_allow_html=True)`), which is unsupported and breaks across versions. If you need pixel-level control or a bespoke design system, Streamlit will frustrate you — that is by design.

## When to Use / When Not

**Use when:**
- You are a Python/data person who wants an interactive UI without touching JS or a web stack.
- Internal tools, ML demos, dashboards, and LLM/chat prototypes for a modest number of concurrent users.
- Speed of building matters more than UI polish or scale.
- You want managed hosting for free via Streamlit Community Cloud.

**Avoid when:**
- You need a consumer-facing product at high concurrency, or fine-grained multi-user auth and roles.
- You need custom, brand-controlled UI/UX or complex client-side interactivity.
- Your workload is CPU-bound per session and must serve many users from shared infrastructure.
- You want a stateless, trivially horizontally-scalable web tier.

## Alternatives

- gradio-app/gradio — similar "Python function to web UI" model, tighter Hugging Face integration; often preferred for ML model demos.
- plotly/dash — callback-graph architecture instead of full reruns; more verbose but scales and customizes better for production dashboards.
- holoviz/panel — more flexible layout/reactivity and works well in Jupyter; steeper learning curve.
- reflex-dev/reflex — compiles Python to a React frontend for real client-side apps when you outgrow the rerun model.
- posit-dev/py-shiny — reactive (not rerun-based) Python UI framework from the makers of R Shiny; better for genuinely reactive dataflow.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2019-10 | Open-sourced; rerun model, core widgets, caching[^1]. |
| 1.0 | 2021-10 | API stability milestone. |
| (acq.) | 2022-03 | Snowflake acquires Streamlit[^1]. |
| 1.10 | 2022-06 | Native multipage apps via `pages/` directory. |
| 1.18 | 2023 | `st.cache_data` / `st.cache_resource` replace `st.cache`[^3]. |
| 1.24 | 2023 | Chat elements (`st.chat_message`, `st.chat_input`) for LLM apps. |
| 1.33 | 2024 | `st.fragment` for partial reruns[^5]. |
| 1.36 | 2024 | `st.navigation` / `st.Page` programmatic routing[^5]. |
| 1.42 | 2025 | Native OIDC auth (`st.login` / `st.logout`)[^7]. |

## References

[^1]: Streamlit — "The fastest way to build data apps," launch and company background; Snowflake acquisition announced March 2022. https://streamlit.io/ and https://www.snowflake.com/blog/welcome-streamlit-snowflake/
[^2]: Streamlit docs, "Basic concepts / App model" — the top-to-bottom rerun execution model. https://docs.streamlit.io/develop/concepts/architecture/run-your-app
[^3]: Streamlit docs, "Caching overview" — `st.cache_data` vs `st.cache_resource`. https://docs.streamlit.io/develop/concepts/architecture/caching
[^4]: Streamlit docs, "Session State." https://docs.streamlit.io/develop/concepts/architecture/session-state
[^5]: Streamlit docs, "Fragments" and "Multipage apps." https://docs.streamlit.io/develop/concepts/architecture/fragments
[^6]: Streamlit docs, "Custom Components." https://docs.streamlit.io/develop/concepts/custom-components/intro
[^7]: Streamlit docs, "User authentication and information." https://docs.streamlit.io/develop/concepts/connections/authentication

## Tags

python, data-apps, dashboard, data-science, machine-learning, web-framework, low-code, rerun-model, apache-2.0, llm-apps
