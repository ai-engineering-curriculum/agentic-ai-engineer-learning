# exercise-02: Agent Long-Term Memory

**Estimated effort:** 3 hours

## Objective

Give an agent the three memory tiers — working, episodic, and long-term — and wire long-term memory as a write-then-recall system with **just-in-time** retrieval and **conflict resolution**. By the end your agent will remember a user's preferences across separate sessions, retrieve them only when a turn needs them, and do the right thing when a new fact contradicts an old one.

## Background

This exercise covers material from:

- [Chapter 2 — Memory Systems for Agents](../02-agent-memory.md)
- [Chapter 3 — Just-in-Time Retrieval and Conflict Resolution](../03-jit-retrieval-conflict.md)

Reuse the vector store and embedding wrapper from [exercise-01](exercise-01-rag-pipeline-vector-db.md) and the agent loop from [mod-201](../../mod-201-agent-fundamentals/README.md). Memory is the same machinery as RAG pointed at a different question: *"what do I already know about this user?"*

## Prerequisites

- The RAG retrieval wrapper from exercise-01 (vector store + embedding).
- An agent loop with tool calling (from mod-201).
- A key-value store can be a dict or a SQLite table; the vector store is your exercise-01 one.

## Tasks

### 1. Episodic memory with summarization

- Implement an episodic memory that keeps the last N turns verbatim and rolls older turns into a running summary.
- Expose a `context()` method that returns the summary plus recent turns for injection into each turn.

### 2. Long-term store: write

- Implement `remember(fact)` that writes a clean, self-contained statement (not a raw transcript) to long-term memory with metadata: `timestamp`, `source` (`user_stated` vs. `agent_inferred`), and a `confidence`.
- Run consolidation as an explicit step: decide *what's worth remembering* (durable preferences/facts) and skip transient chatter.

### 3. Long-term store: recall as a JIT tool

- Expose `recall_memory(cue)` as a **tool** the agent calls, not an eager pre-load. The agent should retrieve memory only when a turn needs it.
- Give the agent a system prompt that says when to recall ("look up what you know about the user when it's relevant; don't recall for things already in this conversation").

### 4. Conflict resolution

- Seed a contradiction: the user said "I'm on the free tier" in an earlier session, then says "I upgraded to pro" now.
- On **recall**, when two memories are about the same attribute and disagree, pick the winner by **source authority, then recency, then confidence** — and inject only the winner (or surface both with metadata and let the model judge).
- On **write**, supersede or delete the stale memory so it never resurfaces.

### 5. Cross-session demonstration

- Run session A: the user states two preferences. End the session (drop working + episodic memory).
- Run session B (fresh process / cleared in-context state): the agent recalls the preferences from long-term memory and uses them without being re-told.

## Starter guidance

```python
from dataclasses import dataclass

@dataclass
class Memory:
    fact: str
    timestamp: str
    source: str       # "user_stated" | "agent_inferred"
    confidence: float

SOURCE_RANK = {"user_stated": 2, "agent_inferred": 1}

def remember(store, fact: str, source: str, confidence: float) -> None:
    raise NotImplementedError  # embed + upsert with metadata; consolidate

def recall_memory(store, cue: str, k: int = 5) -> list[Memory]:
    raise NotImplementedError  # JIT: called by the agent as a tool

def resolve(conflicting: list[Memory]) -> Memory:
    return max(conflicting,
               key=lambda m: (SOURCE_RANK[m.source], m.timestamp, m.confidence))
```

You do **not** need advanced retrieval (exercise-03) or formal evaluation (exercise-04) here.

## Acceptance criteria

You can demonstrate that:

- Episodic memory keeps recent turns verbatim and summarizes older ones; a long conversation does not overflow.
- Long-term writes are clean facts with `timestamp`, `source`, and `confidence` — not transcript dumps.
- `recall_memory` is invoked by the agent **as a tool, on demand** — not pre-loaded on every turn. Show a turn where the agent answers *without* recalling.
- The "free tier" → "pro" contradiction resolves to **pro**, and the stale memory is superseded so it never resurfaces.
- A second, fresh session recalls preferences set in the first session.

## Reflection

In `NOTES.md`:

1. Which facts did your consolidation step *decline* to remember, and why? What's the cost of remembering too much?
2. Show one turn where JIT retrieval saved a recall call versus eager pre-loading, and one where it cost an extra round-trip.
3. Construct a conflict where **recency** gives the wrong answer and **source authority** gives the right one. How did your `resolve` order handle it?

## Stretch goals

- Add an **expiry / decay**: memories below a confidence threshold or past an age are dropped on recall.
- Make consolidation itself an LLM step that reads the session and proposes memories to write, with a human (you) approving the list.
- Combine RAG (exercise-01) and memory in one agent: it answers from the corpus *and* personalizes from long-term memory in the same turn, citing which is which.
