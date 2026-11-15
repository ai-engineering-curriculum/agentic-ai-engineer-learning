# Lab 1 — LangGraph: Stream, Inspect State, Resume

**Duration:** ~45 minutes
**Covers:** [Chapter 2 — LangGraph: Stateful Graph Workflows](../02-langgraph-stateful-graphs.md)

## Objective

Chapter 2 claims three things that are hard to feel from a landing page:

1. The graph runtime emits a *stream* of events you can read as the run progresses, not just a final answer.
2. A **checkpointer** persists state after every node, keyed by `thread_id`, so a second invoke on the same thread continues from where the first left off.
3. `recursion_limit` is a hard, bounded failure — the runtime raises rather than looping forever.

This lab lets you see all three with the shortest possible build. You will use the **prebuilt `create_react_agent`** from `langgraph.prebuilt` — [exercise-01](../exercises/exercise-01-langgraph-stateful-agent.md) is where you drop down to a custom `StateGraph`. Here the point is the *runtime*, not the graph.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- A provider API key with a small spend cap. Examples use `langchain-openai` and `gpt-4o-mini`; swap in any LangChain-supported chat model you prefer.
- Chapter 2 read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install langgraph langchain-openai
mkdir lab-01 && cd lab-01
export OPENAI_API_KEY=...   # your key
```

## Step 1 — A two-tool prebuilt agent in a dozen lines

Save as `agent.py`:

```python
import ast
import operator as op
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import MemorySaver


_OPS = {ast.Add: op.add, ast.Sub: op.sub, ast.Mult: op.mul,
        ast.Div: op.truediv, ast.USub: op.neg}


def _safe_eval(node):
    if isinstance(node, ast.Expression):
        return _safe_eval(node.body)
    if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
        return node.value
    if isinstance(node, ast.BinOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_safe_eval(node.left), _safe_eval(node.right))
    if isinstance(node, ast.UnaryOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_safe_eval(node.operand))
    raise ValueError("disallowed expression")


@tool
def calc(expression: str) -> str:
    """Evaluate a basic arithmetic expression. Supports + - * / and parentheses."""
    return str(_safe_eval(ast.parse(expression, mode="eval")))


@tool
def lookup_population(country: str) -> str:
    """Return a 2024 estimated population for a small fixed set of countries."""
    corpus = {"india": 1_450_000_000, "china": 1_410_000_000,
              "united states": 335_000_000, "indonesia": 280_000_000}
    key = country.strip().lower()
    if key not in corpus:
        return f"unknown country: {country!r}"
    return f"{country}: {corpus[key]:,}"


llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

CHECKPOINTER = MemorySaver()
AGENT = create_react_agent(
    model=llm,
    tools=[calc, lookup_population],
    checkpointer=CHECKPOINTER,
)
```

Deliberate choices:

- `create_react_agent` is the Chapter 2 on-ramp. The loop (model -> tools -> model until no more tool calls) is built in.
- `MemorySaver` is the in-process checkpointer. Chapter 2 recommends SQLite or Postgres for anything you want to survive a restart; `MemorySaver` is the right shape for a lab.
- Both tools are deterministic so you can rerun and get the same trace.

## Step 2 — Stream `values` and read every intermediate state

Save as `stream_values.py`:

```python
from langchain_core.messages import HumanMessage
from agent import AGENT

config = {"configurable": {"thread_id": "demo-values"},
          "recursion_limit": 12}

task = ("Look up the population of India and China, then compute "
        "India divided by China as a ratio.")

for i, snapshot in enumerate(
    AGENT.stream({"messages": [HumanMessage(task)]},
                 config=config,
                 stream_mode="values"),
    start=1,
):
    last = snapshot["messages"][-1]
    kind = type(last).__name__
    preview = (last.content or "").strip().replace("\n", " ")[:80]
    tool_calls = getattr(last, "tool_calls", None)
    tag = f" tool_calls={[tc['name'] for tc in tool_calls]}" if tool_calls else ""
    print(f"[tick {i}] last={kind}{tag} :: {preview!r}")
```

Run:

```bash
python stream_values.py
```

**Expected output (shape, not exact bytes):** six to nine ticks, each printing the *last message* in the full snapshot. You should see something like:

```
[tick 1] last=HumanMessage :: 'Look up the population of India and China, ...'
[tick 2] last=AIMessage tool_calls=['lookup_population'] :: ''
[tick 3] last=ToolMessage :: 'India: 1,450,000,000'
[tick 4] last=AIMessage tool_calls=['lookup_population'] :: ''
[tick 5] last=ToolMessage :: 'China: 1,410,000,000'
[tick 6] last=AIMessage tool_calls=['calc'] :: ''
[tick 7] last=ToolMessage :: '1.0283687943262412'
[tick 8] last=AIMessage :: 'India has roughly 1.03x the population of ...'
```

The model may batch the two lookups into one assistant message with two `tool_calls`, in which case you will see fewer ticks — that is also valid. The point is that *you did not have to add a single `print` inside your loop*; every state transition surfaced on the stream.

**Why this matters:** Chapter 2's `stream_mode="values"` emits the **full state** after each node. The per-tick payloads are bigger than `"updates"` payloads but trivial to render.

## Step 3 — Compare against `stream_mode="updates"`

Save as `stream_updates.py`:

```python
from langchain_core.messages import HumanMessage
from agent import AGENT

config = {"configurable": {"thread_id": "demo-updates"},
          "recursion_limit": 12}

task = "Look up the population of India and divide it by 1,000,000."

for i, update in enumerate(
    AGENT.stream({"messages": [HumanMessage(task)]},
                 config=config,
                 stream_mode="updates"),
    start=1,
):
    for node, payload in update.items():
        msgs = payload.get("messages", [])
        for m in msgs:
            kind = type(m).__name__
            tc = getattr(m, "tool_calls", None)
            tag = f" tool_calls={[t['name'] for t in tc]}" if tc else ""
            preview = (m.content or "").strip().replace("\n", " ")[:80]
            print(f"[tick {i}] node={node} emits {kind}{tag} :: {preview!r}")
```

Run it. Compare to Step 2: each tick now tags which **node** produced the update (`agent` for the model, `tools` for the tool node) and contains only the *new* messages, not the cumulative history. That is the Chapter 2 tradeoff — `updates` is cheaper to ship over a network, but the consumer has to apply the reducers itself to reconstruct the full state.

**Checkpoint:** you can state, without looking, which stream mode emits which payload shape and why a UI showing per-node progress bars would prefer `updates`.

## Step 4 — Inspect the checkpoint with `get_state`

Both prior runs used distinct `thread_id`s, so each ran from scratch. Now inspect what the checkpointer persisted for `demo-values`.

Save as `inspect.py`:

```python
from agent import AGENT

config = {"configurable": {"thread_id": "demo-values"}}
snap = AGENT.get_state(config)

print("values keys:", list(snap.values.keys()))
print("message count:", len(snap.values["messages"]))
for i, m in enumerate(snap.values["messages"]):
    kind = type(m).__name__
    tc = getattr(m, "tool_calls", None)
    tag = f" tool_calls={[t['name'] for t in tc]}" if tc else ""
    print(f"  [{i}] {kind}{tag}")

print("next:", snap.next)
```

Run:

```bash
python inspect.py
```

**Expected output (shape):** a dump of the full `messages` list — `HumanMessage`, `AIMessage(tool_calls=[...])`, `ToolMessage`, `AIMessage(tool_calls=[...])`, `ToolMessage`, (maybe one more `AIMessage(tool_calls=[calc])` + `ToolMessage`), final `AIMessage`. `snap.next` will be an empty tuple `()` because the run completed and there is no pending node.

**Why this matters:** this is the Chapter 2 "human-in-the-loop" primitive without the human. `get_state` is what an approval UI reads to show the operator *what the agent was about to do next*; `graph.update_state` is how the operator overwrites a slot; `graph.invoke(None, config)` is how the operator says "continue."

## Step 5 — Resume on the same `thread_id`

Save as `resume.py`:

```python
from langchain_core.messages import HumanMessage
from agent import AGENT

config = {"configurable": {"thread_id": "demo-values"},
          "recursion_limit": 12}

# First invocation already completed in Step 2. Now send a follow-up on the SAME thread.
followup = "Now also divide that same India number by the population of Indonesia."

final = AGENT.invoke({"messages": [HumanMessage(followup)]}, config=config)

print("final content:")
print(final["messages"][-1].content)
print()
print("message count after follow-up:", len(final["messages"]))
```

Run:

```bash
python resume.py
```

**Expected output:** the agent answers the follow-up *without re-looking-up India*, because the prior `ToolMessage` with India's population is still in `messages`. The `message count after follow-up` is larger than what you printed in Step 4 — the new human message, new assistant message(s), new tool messages, and the final answer have all been appended. Confirm by rerunning `inspect.py`; `snap.values["messages"]` grew.

Change the `thread_id` to anything new, rerun `resume.py`, and watch the agent ask for India again — a fresh thread has no history.

**Why this matters:** Chapter 2's one-liner — *compile once, invoke many; each `invoke` with a distinct `thread_id` is an independent conversation* — is load-bearing for every production deployment. Your `thread_id` strategy is your conversation model.

## Step 6 — Hit the `recursion_limit`

Save as `blow_the_budget.py`:

```python
from langchain_core.messages import HumanMessage
from langgraph.errors import GraphRecursionError
from agent import AGENT

config = {"configurable": {"thread_id": "demo-blow"},
          "recursion_limit": 3}   # absurdly low on purpose

task = ("Look up the population of India, then of China, then of the "
        "United States, then of Indonesia, then compute India/China, "
        "then India/United States. Show your work.")

try:
    AGENT.invoke({"messages": [HumanMessage(task)]}, config=config)
except GraphRecursionError as e:
    print("bounded failure:", type(e).__name__)
    print("message:", str(e).splitlines()[0])
```

Run:

```bash
python blow_the_budget.py
```

**Expected output:** a `GraphRecursionError` message. The runtime *bounded* the failure — the process did not spin forever, and the exception is distinct from a tool error or an HTTP timeout.

**Why this matters:** one ReAct iteration is two node transitions (`agent -> tools` and `tools -> agent`), so `recursion_limit=3` cannot even complete one full iteration. The lesson from Chapter 2: `recursion_limit` is a **step budget**, not an iteration budget, and the default is low enough that real agents hit it — set it per invoke, deliberately.

## Step 7 — Adversarial checkpoint: swap the checkpointer

In `agent.py`, replace `MemorySaver()` with no checkpointer at all:

```python
AGENT = create_react_agent(
    model=llm,
    tools=[calc, lookup_population],
    # checkpointer removed
)
```

Rerun `resume.py` with the same `thread_id="demo-values"`. The follow-up now has no history to resume from — the agent re-looks-up India from scratch and may never answer Indonesia unless you re-include the prior answer in the new `HumanMessage`.

Put the `MemorySaver` back.

**Checkpoint:** you can state the exact failure mode of "forgot to compile with a checkpointer" (every invoke is a cold start) and the exact failure mode of "wrong `thread_id`" (cold start even *with* a checkpointer).

## Wrap-up

You now have, in under a hundred lines of code, direct hands-on evidence for every Chapter 2 claim about the runtime:

- `stream` emits per-node events in real time — your UI can render them without changes to the loop.
- `values` carries the full state after each node; `updates` carries only the delta. Pick by consumer cost.
- `get_state(config)` gives you the pending state, which is the hook human approval UIs read.
- `thread_id` + checkpointer is the entire resumption story; swap in `SqliteSaver` for a durable variant without changing the rest of the code.
- `recursion_limit` is the bounded failure — set it per invoke; the default is low enough to bite you.

[Exercise 01](../exercises/exercise-01-langgraph-stateful-agent.md) is where you drop below the prebuilt and write a custom `StateGraph` with a typed state, a non-message slot, a conditional edge, and `interrupt_before`. Everything you measured here still applies — the prebuilt agent *is* a `StateGraph` under the hood.
