# exercise-04: HITL with Persistence

**Estimated effort:** 3 hours

## Objective

Add a human-in-the-loop approval gate to an agent so it pauses before a consequential action, persists its full state to a durable store, and resumes — from a *different process* — once a human approves, rejects, or edits. By the end you'll have a paused run survive a restart and resume exactly where it stopped, with the gated action firing exactly once.

## Background

This exercise covers material from:

- [Chapter 4 — Human-in-the-Loop with State Persistence](../04-hitl-with-persistence.md)
- [Chapter 2 — Durable Execution and Resumption](../02-durable-execution.md) — for the idempotency discipline on the gated action.

Use LangGraph's checkpointer + `interrupt()` for a concrete implementation. The agent only needs one consequential step (sending an email, posting a comment, issuing a refund — stub the actual effect).

## Prerequisites

- An agent with at least one consequential action to gate.
- LangGraph installed, plus a database-backed checkpointer (Postgres or SQLite — **not** the in-memory one for the persistence task).
- API key in an environment variable; small spend cap.

## Tasks

### 1. Build the graph with an approval node

- Model the agent as a LangGraph graph where the consequential step is preceded by an `approval_node` that calls `interrupt(...)`.
- The interrupt payload must carry enough for a human to decide: the proposed action, the reasoning, and the relevant inputs.

### 2. Persist with a database-backed checkpointer

- Compile the graph with a Postgres- or SQLite-backed checkpointer, keyed by a `thread_id`.
- Run until the interrupt and confirm control returns to the caller with the run paused.

### 3. Resume across a process boundary

- In a **separate process** (or after restarting your script), load the same `thread_id` and resume with `Command(resume=...)` carrying the decision.
- Prove the agent continues from the interrupt — not from the start — using the persisted state.

### 4. Approve, reject, and edit

- Support all three resume outcomes: approve (proceed), reject (skip the action, end cleanly), and edit (the human modifies the draft and the agent uses the edited version).

### 5. Idempotent gated action

- Make the gated side effect idempotent via an idempotency key tied to the `thread_id`.
- Resume the same run twice (simulating a double-approval) and prove the external action fires exactly once.

## Starter guidance

```python
from langgraph.graph import StateGraph, END
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.types import interrupt, Command

def approval_node(state: dict) -> dict:
    decision = interrupt({
        "action": state["proposed_action"],
        "reason": state["reason"],
        "draft": state["draft"],
    })
    if not decision["approved"]:
        return {"status": "rejected"}
    return {"status": "approved", "draft": decision.get("edited_draft", state["draft"])}

def act_node(state: dict) -> dict:
    if state["status"] != "approved":
        return {"result": "skipped"}
    key = f'{state["thread_id"]}:send'
    send_once(key, state["draft"])  # idempotent on key
    return {"result": "sent"}

# build graph: ... -> approval_node -> act_node -> END
app = graph.compile(checkpointer=PostgresSaver(conn))
config = {"configurable": {"thread_id": "run-77"}}

app.invoke({"proposed_action": "email customer", "draft": "..."}, config)  # pauses
# ...later, different process...
app.invoke(Command(resume={"approved": True}), config)                    # resumes
```

You do **not** need Temporal here, though the persistence discipline is the same; combining the two is a stretch goal.

## Acceptance criteria

You can demonstrate that:

- The agent pauses at the `interrupt` and returns control with the run persisted under its `thread_id`.
- A **different process** resumes the run from persisted state and continues from the interrupt, not the start.
- Approve, reject, and edit all work, and edit causes the agent to use the modified draft.
- The gated action is idempotent: resuming twice fires the external effect exactly once.
- Killing the process while paused and restarting it loses nothing — the run still resumes.

## Reflection

In `NOTES.md`:

1. What state had to be persisted for the resume to be correct? What would have broken with an in-memory checkpointer?
2. How did you make the gated action idempotent, and which double-fire scenario were you defending against?
3. Compare this HITL wait to the durable-execution wait from exercise-02. What's shared, and what's different about *why* each one pauses?

## Stretch goals

- Add an authorization check so only a permitted approver can resume the run, and reject an unauthorized resume.
- Add a timeout policy: if no decision arrives within a window, escalate or default-deny, and persist that outcome.
- Run the LangGraph agent inside a Temporal workflow (exercise-02) so the human approval is a workflow signal and the whole run is both durable and gated.
