# exercise-01: LangGraph Stateful Agent

**Estimated effort:** 3 hours

## Objective

Build the canonical research-assistant agent as a LangGraph `StateGraph`: a typed state object, a model node, a tool node, a conditional edge that implements the ReAct termination policy, and a checkpointer that makes the run resumable. By the end you'll know exactly which parts of your hand-rolled mod-201 loop LangGraph is replacing and which it leaves to you.

## Background

This exercise covers material from:

- [Chapter 1 — The Framework Landscape](../01-the-framework-landscape.md)
- [Chapter 2 — LangGraph: Stateful Graph Workflows](../02-langgraph-stateful-graphs.md)

Use any LangChain-supported model provider. A real web-search tool makes the demo task realistic, but you may stub `search` with a fixed corpus so the exercise runs offline and deterministically.

## Prerequisites

- The reason-act loop and tool calling from [mod-201](../../mod-201-agent-fundamentals/README.md).
- `langgraph` and a `langchain-*` provider package installed.
- A provider API key in an environment variable; a small spend cap.

## Tasks

### 1. Define the typed state

- Declare an `AgentState` `TypedDict` with a `messages` slot annotated with `add_messages`.
- Add at least one non-message slot that a node writes and a later node reads — for example a `step` counter with an `operator.add` reducer, or a `budget` slot. The point is to use a reducer other than the message one.

### 2. Build the model and tool nodes

- Define two tools: `search(query: str) -> str` (stub or real) and `calc(expression: str) -> str` using a safe arithmetic evaluator (`ast`-based, **not** `eval`).
- Bind the tools to the model with `bind_tools`.
- Write `model_node(state)` returning `{"messages": [response]}`.
- Use the prebuilt `ToolNode(tools)` for tool execution rather than hand-writing a dispatch dict.

### 3. Wire the graph and the termination policy

- Add `model` and `tools` nodes; edge `START -> model`; edge `tools -> model`.
- Add a conditional edge from `model` via a `should_continue(state)` router that returns `"tools"` when the last message has `tool_calls`, otherwise `END`.
- Set an explicit `recursion_limit` per invoke so a misbehaving loop terminates.

### 4. Add a checkpointer and prove resumption

- Compile with a `MemorySaver` (or `SqliteSaver` for a durable variant).
- Invoke once with a `thread_id`, then invoke again on the **same** `thread_id` with a follow-up and show the prior state carried over.

### 5. Add a human-in-the-loop interrupt

- Recompile with `interrupt_before=["tools"]`.
- On interrupt, read the pending state with `get_state`, print the proposed tool call, and resume with `graph.invoke(None, config)` after a simulated approval.

## Starter guidance

```python
import ast
import operator as op
from typing import Annotated, TypedDict

from langchain_core.messages import BaseMessage, HumanMessage
from langchain_core.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver


class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    step: Annotated[int, op.add]


@tool
def search(query: str) -> str:
    """Search a fixed corpus and return the best snippet."""
    raise NotImplementedError  # return a deterministic stub for offline runs


@tool
def calc(expression: str) -> str:
    """Evaluate a basic arithmetic expression safely (no eval)."""
    raise NotImplementedError  # ast.parse + a whitelist of node types


def model_node(state: AgentState) -> dict:
    raise NotImplementedError  # call the tool-bound model, return {"messages": [...], "step": 1}


def should_continue(state: AgentState) -> str:
    raise NotImplementedError  # "tools" if last message has tool_calls else END


def build_graph():
    builder = StateGraph(AgentState)
    builder.add_node("model", model_node)
    builder.add_node("tools", ToolNode([search, calc]))
    builder.add_edge(START, "model")
    builder.add_conditional_edges("model", should_continue, {"tools": "tools", END: END})
    builder.add_edge("tools", "model")
    return builder.compile(checkpointer=MemorySaver())


if __name__ == "__main__":
    graph = build_graph()
    config = {"configurable": {"thread_id": "demo-1"}, "recursion_limit": 12}
    out = graph.invoke(
        {"messages": [HumanMessage("Compare the populations of the two most "
                                   "populous countries and report the ratio.")],
         "step": 0},
        config=config,
    )
    print(out["messages"][-1].content)
```

## Acceptance criteria

You can demonstrate that:

- The agent completes the population-ratio task with the expected trajectory (two searches, one `calc`, one final answer) without hand-written dispatch.
- The non-message state slot is written by a node and read by another, using a reducer other than `add_messages`.
- A second invoke on the same `thread_id` continues from the prior checkpoint rather than starting fresh.
- With `interrupt_before=["tools"]`, the run pauses, exposes the pending tool call via `get_state`, and resumes correctly after a `graph.invoke(None, config)`.
- Hitting the `recursion_limit` raises a bounded error rather than looping forever.

## Reflection

In `NOTES.md`:

1. Which parts of your mod-201 loop did `ToolNode` and the conditional edge replace, and which did you still have to write yourself?
2. What is the smallest change to your graph that would make this a plan-and-execute agent instead of plain ReAct?
3. If you put a 2 MB document into a state slot, what breaks, and where would you store it instead?

## Stretch goals

- Swap `MemorySaver` for `SqliteSaver` and kill the process mid-run; resume from the database on restart.
- Add a third node — a `summarize` node that runs only when `step` exceeds a threshold — and route to it with a conditional edge.
- Stream the run with `stream_mode="updates"` and render node-by-node progress instead of waiting for the final answer.
- Replace the custom graph with `create_react_agent` and write down exactly which of your acceptance criteria you can no longer meet.
