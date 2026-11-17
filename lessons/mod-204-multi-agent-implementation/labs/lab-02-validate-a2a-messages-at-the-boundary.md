# Lab 2 — Type and Validate A2A Messages at the Boundary

**Duration:** ~40 minutes
**Covers:** [Chapter 3 — Inter-Agent Communication: A2A and MCP](../03-inter-agent-communication.md)

## Objective

Chapter 3 argues that the interface — not the agents — is where multi-agent systems rot, and the discipline that prevents the rot is **typed messages validated at the boundary**. None of the module's exercises cover Chapter 3 head-on, so this lab is its hands-on companion. In 40 minutes you will:

- Implement the `AgentMessage` schema from Chapter 3 as a pydantic model.
- Validate messages at the point they cross between agents and see what a bad message looks like.
- Correlate a fan-out's results back to its requests using `task_id`.
- Branch on `intent` so an `error` message is handled as a first-class case, not as a hallucinated success.

No API calls, no model usage. Pure pydantic and dispatch logic.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 3](../03-inter-agent-communication.md) read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install pydantic
mkdir lab-02 && cd lab-02
```

## Step 1 — The `AgentMessage` contract

Translate Chapter 3's example into a strict pydantic model. We use `extra="forbid"` so an agent cannot sneak extra fields past the boundary.

Save as `contract.py`:

```python
from pydantic import BaseModel, ConfigDict, Field
from typing import Literal


class AgentMessage(BaseModel):
    model_config = ConfigDict(extra="forbid")

    sender: str
    recipient: str
    intent: Literal["request", "result", "error"]
    task_id: str
    payload: dict
    confidence: float | None = Field(default=None, ge=0.0, le=1.0)
```

Four details worth noticing:

- `intent` is a `Literal` — the schema itself rejects `"complete"`, `"done"`, `"success"`, and every other near-miss.
- `error` is one of the three intents, by construction. A worker that fails cannot "forget to mark" its message.
- `confidence` is optional *and* bounded to `[0, 1]`. Chapter 3: carry it only if agents calibrate honestly, and only in that range.
- `extra="forbid"` means the validator throws on unknown fields — exactly the "validate at the boundary" behavior Chapter 3 asks for.

## Step 2 — Validate a well-formed message

Save as `validate.py`:

```python
from contract import AgentMessage

good = {
    "sender": "worker-aws",
    "recipient": "orchestrator",
    "intent": "result",
    "task_id": "t-42",
    "payload": {"answer": "EKS supports L4/L7 autoscaling.",
                "sources": ["https://example.test/eks/1"]},
    "confidence": 0.8,
}

msg = AgentMessage.model_validate(good)
print("validated:", msg.model_dump())
```

Run:

```bash
python validate.py
```

**Expected output:** a dict identical to `good`, printed back. That's the boring, correct case — the contract passes untouched.

## Step 3 — Watch what a malformed message looks like

Append to `validate.py`:

```python
from pydantic import ValidationError

bad_cases = [
    # Unknown intent — Chapter 3 says downstream agents branch on intent,
    # so this is exactly the kind of drift the schema must catch.
    {**good, "intent": "complete"},

    # Missing task_id — fan-outs can no longer match results to requests.
    {k: v for k, v in good.items() if k != "task_id"},

    # Extra field — a sender tried to smuggle extra context in.
    {**good, "transcript": "...the whole scratch work..."},

    # Confidence out of range — honest calibration or drop it.
    {**good, "confidence": 1.7},
]

for i, b in enumerate(bad_cases, 1):
    try:
        AgentMessage.model_validate(b)
    except ValidationError as e:
        print(f"[case {i}] rejected: {e.errors()[0]['type']} on {e.errors()[0]['loc']}")
```

Rerun. **Expected output:**

```
validated: {...}
[case 1] rejected: literal_error on ('intent',)
[case 2] rejected: missing on ('task_id',)
[case 3] rejected: extra_forbidden on ('transcript',)
[case 4] rejected: less_than_equal on ('confidence',)
```

**Why this matters:** every rejection here is a failure mode Chapter 3 names. Without validation, each would have propagated as corrupt input to the next agent — intent drift, lost correlation, smuggled transcripts, and miscalibrated confidence becoming a load-bearing number.

## Step 4 — A dispatcher that trusts only validated input

Chapter 3: "Treat another agent's output as untrusted input — validate it against a schema before acting on it, exactly as you'd validate a user's input."

Save as `dispatcher.py`:

```python
from pydantic import ValidationError
from contract import AgentMessage


def dispatch(raw: dict) -> AgentMessage:
    """Validate at the boundary. Anything that fails becomes an `error` message
    the caller can route on — never a raised exception that crashes the loop."""
    try:
        return AgentMessage.model_validate(raw)
    except ValidationError as e:
        return AgentMessage(
            sender="runtime",
            recipient=raw.get("recipient", "unknown") or "unknown",
            intent="error",
            task_id=raw.get("task_id", "unknown") or "unknown",
            payload={"error": "schema_violation", "detail": e.errors()},
        )


def handle(msg: AgentMessage) -> str:
    """Branch on `intent`, not on parsing prose."""
    if msg.intent == "request":
        return f"accepted request {msg.task_id} from {msg.sender}"
    if msg.intent == "result":
        return f"stored result for {msg.task_id}: {msg.payload.get('answer', '')[:60]}"
    if msg.intent == "error":
        return f"error on {msg.task_id}: {msg.payload.get('error', 'unspecified')}"
    raise AssertionError(f"unreachable intent: {msg.intent}")
```

Notice the key move: a malformed message does not raise. It is **repackaged as an `error` intent** with the detail in the payload. The caller never has to decide whether a failed validation is "a result or an error" — the schema already made that choice.

## Step 5 — Correlate a fan-out with `task_id`

Chapter 3 says `task_id` is what lets a fan-out's results be matched back to their assignments. Simulate a mini orchestrator-worker exchange.

Save as `correlate.py`:

```python
from contract import AgentMessage
from dispatcher import dispatch, handle

# Three outgoing requests from the orchestrator.
outgoing = [
    AgentMessage(sender="orch", recipient=f"worker-{w}", intent="request",
                 task_id=f"t-{w}", payload={"instruction": f"Profile {w}"})
    for w in ("aws", "gcp", "azure")
]

# Three replies come back in a different order (that's the point of task_id).
replies_raw = [
    {"sender": "worker-gcp", "recipient": "orch", "intent": "result",
     "task_id": "t-gcp", "payload": {"answer": "GKE autoscales well"}, "confidence": 0.7},
    {"sender": "worker-azure", "recipient": "orch", "intent": "error",
     "task_id": "t-azure", "payload": {"error": "source_timeout"}},
    {"sender": "worker-aws", "recipient": "orch", "intent": "result",
     "task_id": "t-aws", "payload": {"answer": "EKS is fine"}, "confidence": 0.8},
    # A late reply with a task_id we never sent — must not corrupt the index.
    {"sender": "worker-ghost", "recipient": "orch", "intent": "result",
     "task_id": "t-ghost", "payload": {"answer": "?"}},
]

expected = {r.task_id for r in outgoing}
by_task: dict[str, AgentMessage] = {}

for raw in replies_raw:
    msg = dispatch(raw)
    if msg.task_id not in expected:
        print(f"ignored unexpected task_id: {msg.task_id}")
        continue
    by_task[msg.task_id] = msg

for req in outgoing:
    reply = by_task.get(req.task_id)
    if reply is None:
        print(f"{req.task_id}: still pending")
    else:
        print(f"{req.task_id}: {handle(reply)}")
```

Run:

```bash
python correlate.py
```

**Expected output (order may vary slightly):**

```
ignored unexpected task_id: t-ghost
t-aws: stored result for t-aws: EKS is fine
t-gcp: stored result for t-gcp: GKE autoscales well
t-azure: error on t-azure: source_timeout
```

**Why this matters:**

- Replies arrived in a different order from the requests. `task_id` keyed each one back correctly.
- The `t-ghost` reply — a hallucinated message about a task we never spawned — was filtered at the boundary, not accepted into the index.
- The Azure failure surfaced as a structured `error` on the right `task_id`, so the orchestrator knows *which* sub-task is missing coverage. (Compare this to a worker returning a hallucinated success — the orchestrator would silently include bogus Azure data in its synthesis.)

## Step 6 — Break the boundary and feel the difference

Comment out the `msg = dispatch(raw)` indirection in `correlate.py` and replace it with `msg = AgentMessage(**raw)`. Add one more malformed reply to `replies_raw`:

```python
{"sender": "worker-aws", "recipient": "orch", "intent": "complete",
 "task_id": "t-aws", "payload": {}},
```

Rerun. Now a single bad message crashes the entire correlation loop with a `ValidationError`. Any good replies after the crash are lost. **That** is the failure mode Chapter 3 is warning against when it says "validate at the boundary" — without it, one agent's sloppy output eats everyone else's work.

Restore the `dispatch()` indirection when done.

**Checkpoint:** you can state in your own words:

1. Why `intent` is a `Literal`, not a free-form string.
2. What happens to a malformed message under `dispatch` — and why re-packaging it as an `error` intent is strictly safer than raising.
3. Why `task_id` is the only reason a fan-out's out-of-order replies can be reassembled correctly.

## Wrap-up

The contract, the dispatcher, and the task-id keying from this lab are the pieces Exercise 02's stretch goal asks for ("replace the freeform `summary` string with a typed `AgentMessage` and validate it on receipt"). More importantly, every multi-agent system you build after this point should start with this shape: define the message, forbid extras, branch on `intent`, correlate on `task_id`. The interface *is* the hard part — this is what makes it stop being hard.
