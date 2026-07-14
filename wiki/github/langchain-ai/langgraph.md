# langchain-ai/langgraph

> Low-level, graph-based orchestration for stateful, long-running LLM agents — you wire the control flow yourself and get durable execution in return.

[GitHub repo](https://github.com/langchain-ai/langgraph) ·
[Official website](https://docs.langchain.com/oss/python/langgraph/) ·
[License: MIT](https://github.com/langchain-ai/langgraph/blob/main/LICENSE)

## Overview

LangGraph is an agent orchestration library from LangChain Inc, first released in early 2024[^1]. Where the parent `langchain` library gives you high-level chains and prebuilt agents, LangGraph drops one level down: you model your agent as a directed graph of nodes (Python functions) and edges (control-flow transitions) operating over a shared, typed state object. It exists because the ReAct-style "agent executor" loop in early LangChain was hard to inspect, hard to resume, and hard to insert humans into — LangGraph re-expresses the loop as an explicit state machine you can checkpoint, pause, branch, and replay.

The defining tension is verbosity versus control. A "hello world" agent in a higher-level framework is three lines; the same thing in raw LangGraph is a `StateGraph`, a `TypedDict` state schema, node functions, edge wiring, and a `.compile()` call. In exchange you get durable execution (an agent that survives a process crash and resumes from the last checkpoint), first-class human-in-the-loop interrupts, and complete visibility into every state transition. LangGraph is aimed at teams who outgrew the prebuilt loop and need to own the control flow — not at people who want an agent in five minutes.

The second thing to understand up front is commercial gravity. LangGraph the library is MIT-licensed and runs standalone, but it is the on-ramp to a paid stack: LangSmith for tracing/evals, and LangSmith Deployment (formerly LangGraph Platform/Cloud) for hosting long-running graphs. The library is genuinely usable without any of it, but much of the documentation and the "production-ready" story routes through LangChain's hosted products[^2].

## Getting Started

```bash
pip install -U langgraph langchain-openai
```

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_openai import ChatOpenAI

class State(TypedDict):
    # add_messages is a reducer: node returns are appended, not overwritten
    messages: Annotated[list, add_messages]

llm = ChatOpenAI(model="gpt-4o")

def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}

builder = StateGraph(State)
builder.add_node("chatbot", chatbot)
builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)
graph = builder.compile()

result = graph.invoke({"messages": [{"role": "user", "content": "hi"}]})
print(result["messages"][-1].content)
```

For a full tool-calling agent, the `create_react_agent` prebuilt saves the boilerplate:

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(model="openai:gpt-4o", tools=[my_tool])
agent.invoke({"messages": [{"role": "user", "content": "..."}]})
```

## Architecture / How It Works

LangGraph's runtime is an implementation of the **Pregel** message-passing model[^3] (Google's bulk-synchronous graph engine), with API surface borrowed from NetworkX. Execution proceeds in discrete "super-steps": in each step, all active nodes run (potentially in parallel), their outputs are written to state, and the engine decides which nodes activate next. This BSP structure is why LangGraph can checkpoint cleanly — a checkpoint is just the state snapshot at a super-step boundary.

Core primitives:

- **State + reducers.** State is a `TypedDict` (or Pydantic model / dataclass). Each field can carry a reducer function via `Annotated[..., reducer]`. Without a reducer, a node's return value overwrites the field; with one (like `add_messages`), returns are merged. Getting reducers wrong is the most common source of "my state disappeared" bugs.
- **Nodes and edges.** Nodes are functions `state -> partial_state`. Edges are either static (`add_edge`) or **conditional** (`add_conditional_edges`, a router function returning the next node name). Cycles are legal and are how agent loops are expressed.
- **`Command`.** A node can return a `Command` object to update state *and* direct control flow in one move — the modern way to build handoffs between subagents without a separate router node.
- **Checkpointers.** The persistence layer. `MemorySaver` (in-process), `SqliteSaver`, and `PostgresSaver` (in `langgraph-checkpoint-*` packages) snapshot state after each super-step, keyed by a `thread_id`. This is what makes execution durable and time-travel/replay possible.
- **`interrupt()`.** Pauses the graph mid-node, persists state, and returns control to the caller for human approval or edits; resuming replays from the interrupt point. This is the human-in-the-loop mechanism, and it only works with a checkpointer configured.
- **Streaming.** Multiple modes (`values`, `updates`, `messages`, `custom`) let you stream either full state, per-node deltas, or token-level LLM output.

The graph is static once compiled — you cannot add nodes at runtime — but conditional edges and `Command` give you dynamic *paths* through a fixed topology.

## Production Notes

**The checkpointer is not optional in production.** Durable execution, interrupts, and memory all require one, and `MemorySaver` is explicitly not for production (it is lost on restart). Real deployments use `PostgresSaver`; run its `.setup()` migration before first use, and budget for the write volume — a checkpoint is written every super-step, so a chatty multi-turn agent generates a lot of Postgres traffic. Old checkpoints are not garbage-collected automatically; you own retention.

**Version churn.** LangGraph moved fast through 0.x and reached 1.0 in late 2025[^4]. The pre-1.0 period saw repeated API shifts — checkpoint libraries were split out into separate packages in 0.2, and prebuilt/agent APIs were reshaped more than once. Pin exact versions of `langgraph` *and* the `langchain-*` integration packages together; mismatches between LangGraph and the underlying LangChain core surface as confusing import or message-format errors.

**State serialization footguns.** Everything in state must round-trip through the checkpointer's serializer. Putting non-serializable objects (open DB connections, clients, file handles) in state will fail at checkpoint time, not at assignment time — a delayed error. Keep state to plain data; pass live resources via config or closures.

**Debugging is only tolerable with tracing.** A cyclic graph with conditional edges is genuinely hard to reason about from logs alone. LangSmith tracing is the intended debugger and the practical answer for non-trivial graphs; expect to wire it in (`LANGSMITH_TRACING=true`) early. This is real vendor pull, even though the library itself is open.

**Parallelism and reducers.** When multiple nodes write the same state key in the same super-step (fan-out), the reducer must be commutative/associative or results become order-dependent. This is easy to get wrong with a naive "overwrite" field.

**Async and sync are separate paths.** Nodes can be sync or async, and checkpointers come in sync/async variants (`PostgresSaver` vs `AsyncPostgresSaver`). Mixing them (async graph, sync checkpointer) is a recurring source of runtime surprises.

## When to Use / When Not

**Use when:**
- You need long-running or resumable agents that survive crashes and pick up where they left off.
- Human-in-the-loop approval, editing, or interruption is a hard requirement.
- Your control flow is genuinely graph-shaped: branching, cycles, multi-agent handoffs, subgraphs.
- You want explicit, inspectable state transitions rather than an opaque agent loop.

**Avoid when:**
- You want a simple tool-calling chatbot — the prebuilt `create_react_agent`, or a lighter framework, gets you there with far less code.
- You want to avoid the LangChain ecosystem's abstractions and dependency surface; LangGraph inherits a lot of it.
- Your workload is stateless request/response with no durability, HITL, or complex branching needs.
- You need a stable, slow-moving API — the library has iterated aggressively and upgrades have cost real migration time.

## Alternatives

- microsoft/autogen — use instead when you prefer a conversation-centric multi-agent model over explicit graph wiring.
- crewAIInc/crewAI — use instead when you want higher-level role/task abstractions and less control-flow plumbing.
- openai/openai-agents-python — use instead when you want a lighter, handoff-based agent SDK without the durable-execution machinery.
- pydantic/pydantic-ai — use instead when type-safety and a Pydantic-native design matter more than graph orchestration.
- langchain-ai/langchain — use instead (or first) when the prebuilt agent loop is enough and you don't need custom control flow.

## History

| Version | Date | Notes |
|---------|------|-------|
| announced | 2024-01 | LangGraph introduced as LangChain's stateful/multi-actor agent layer[^1]. |
| 0.1 | 2024-06 | First 0.1 line; stabilized `StateGraph` API. |
| 0.2 | 2024-08 | Checkpointer libraries split into separate `langgraph-checkpoint-*` packages. |
| 0.3–0.4 | 2025 | `Command`-based control flow, reshaped prebuilts, functional API. |
| 1.0 | 2025-10 | First stable major release[^4]. |

## References

[^1]: LangChain blog, "LangGraph" announcement — Jan 2024. https://blog.langchain.dev/langgraph/
[^2]: LangGraph documentation (LangChain OSS docs). https://docs.langchain.com/oss/python/langgraph/overview
[^3]: G. Malewicz et al., "Pregel: A System for Large-Scale Graph Processing" — Google, 2010. https://research.google/pubs/pub37252/
[^4]: LangChain, "LangChain and LangGraph 1.0" release notes. https://docs.langchain.com/oss/python/releases/langgraph-v1

## Tags

python, ai-agents, llm, orchestration, agent-framework, state-machine, langchain, durable-execution, human-in-the-loop, multi-agent, workflow-engine
