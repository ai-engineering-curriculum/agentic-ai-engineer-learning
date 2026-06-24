# Chapter 4 — Human-in-the-Loop with State Persistence

Some agent actions are too consequential to auto-execute: sending an external email, merging a PR, issuing a refund, deleting data. The production answer is a **human-in-the-loop (HITL)** gate — the agent pauses at the risky step, a person approves or edits, and the agent resumes. The hard part isn't the pause; it's that the pause can last minutes or days, and the agent's full state has to survive it, possibly across a process restart or a different server.

## The pattern: checkpoint, interrupt, resume

You need three things:

1. **A persisted checkpoint** of the agent's state (messages, scratchpad, where it is in the graph) written to a durable store, keyed by a thread/run id.
2. **An interrupt** that suspends execution at the gate and surfaces what needs approval to the outside world.
3. **A resume** that loads the checkpoint, injects the human's decision, and continues from exactly that point.

LangGraph models this directly. A **checkpointer** saves graph state after every step; `interrupt()` pauses a node and bubbles a payload out to the caller; resuming with a `Command(resume=...)` re-enters the graph at the interrupt with the human's input.

```python
from langgraph.graph import StateGraph
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.types import interrupt, Command

def approval_node(state: dict) -> dict:
    decision = interrupt({
        "action": state["proposed_action"],
        "reason": state["reason"],
    })
    if decision["approved"]:
        return {"status": "approved", "edits": decision.get("edits")}
    return {"status": "rejected"}

graph = build_graph(approval_node)            # wire your nodes
app = graph.compile(checkpointer=PostgresSaver(conn))

config = {"configurable": {"thread_id": "run-42"}}

# First call runs until the interrupt, then returns control.
app.invoke({"proposed_action": "email customer"}, config)

# ...minutes or days later, possibly in a different process...
app.invoke(Command(resume={"approved": True}), config)
```

Because state lives in the checkpointer (Postgres here, not memory), the second `invoke` can run in a *different process* than the first. That's what makes HITL production-grade: the human's wait doesn't pin a server open, and a deploy in the middle doesn't lose the run.

## Design rules for the gate

- **Persist to a real store, not memory.** An in-memory checkpointer demos fine and loses every paused run on restart. Use a database-backed checkpointer in production.
- **Surface enough to decide.** The interrupt payload must carry exactly what the approver needs — the proposed action, the reasoning, the inputs — so they can judge without re-reading the whole transcript.
- **Support approve, reject, *and* edit.** Real reviewers don't just say yes/no; they tweak the draft. Let the resume value carry edits the agent then uses.
- **Make the gated action idempotent.** A resume might be retried (double-clicked approval, a redelivered webhook). The downstream side effect needs an idempotency key so it fires once — the same discipline as durable activities ([Chapter 2](02-durable-execution.md)).
- **Time out and authorize.** A run can wait forever; decide what happens on no response (escalate, expire, default-deny). And check that the approver is *allowed* to approve — the gate is a security boundary, not just a UX step.

## How it relates to durable execution

Durable execution ([Chapter 2](02-durable-execution.md)) and HITL persistence are the same idea aimed at two problems. Both persist state so a run survives a gap; durable execution does it for *crashes*, HITL does it for *deliberate human waits*. They stack: a Temporal workflow can wait on a signal that is exactly a human approval, and a LangGraph agent can run inside a durable workflow. The shared discipline — persist the state, make the resumed side effect idempotent — is the whole game.

## Key takeaways

- HITL gates pause the agent before consequential actions; the engineering challenge is surviving a wait that can span restarts and machines.
- The pattern is checkpoint → interrupt → resume: LangGraph's checkpointer persists state, `interrupt()` pauses and surfaces the decision, `Command(resume=...)` continues from the exact point.
- Persist to a database-backed store, surface enough context to decide, and support approve/reject/edit.
- Make the gated action idempotent and authorize the approver — the gate is a correctness *and* security boundary, the same persistence discipline as durable execution.
