# Chapter 4 — Sub-Agent Isolation and Distilled Returns

The reason multi-agent systems work is not that "more agents are smarter." It's **context economics**: a sub-agent runs in its *own* context window, burns tokens there, and returns a small distilled result — so the orchestrator's context stays clean no matter how much work the sub-agent did.

## The core idea

A sub-agent is isolated when:

1. It starts with **only its assignment** in context — not the orchestrator's conversation, not its siblings' work.
2. It does its work — many tool calls, large documents, long reasoning — entirely **inside its own window**.
3. It returns a **distilled result**: the answer, not the transcript.

```
Orchestrator context window (stays small)
  ├─ assignment for worker A ──▶ [Worker A: reads 30 docs, 18 tool calls] ──▶ "summary A" (300 tokens)
  ├─ assignment for worker B ──▶ [Worker B: …] ──────────────────────────▶ "summary B" (300 tokens)
  └─ synthesize(summary A, summary B)
```

Worker A might consume 80K tokens internally. The orchestrator only ever sees 300. That asymmetry is the whole point — it's how you investigate something huge without the lead agent drowning in raw material.

## Distilled returns: summary, not transcript

The most common multi-agent anti-pattern is a worker returning its entire scratch work — every tool result, every intermediate thought. That defeats isolation: the orchestrator's context fills with exactly the noise you spun up a sub-agent to avoid.

A distilled return is **the deliverable plus just enough provenance to trust it**:

```python
class WorkerResult(BaseModel):
    answer: str                 # the actual finding/output
    sources: list[str]          # ids/urls/paths that back it — references, not contents
    confidence: Literal["high", "medium", "low"]
    open_questions: list[str]   # what it could NOT resolve
```

Tell the worker, in its system prompt, to return this shape: *"Report your finding and the sources that support it. Do not include your search process or raw tool output."* Now the orchestrator gets a 300-token result it can act on, with pointers to drill into if it needs to.

## Implementing isolation

The key is that each sub-agent gets a **fresh message list** seeded only with its task — never the parent's history:

```python
async def run_isolated(role: str, assignment: str) -> WorkerResult:
    messages = [{"role": "user", "content": assignment}]   # fresh — no parent context
    result = await agent_loop(
        system=WORKER_SYSTEM[role],
        messages=messages,
        tools=TOOLS_FOR[role],            # scoped per role (least privilege)
        response_model=WorkerResult,      # force the distilled shape
    )
    return result                          # only this crosses back to the orchestrator
```

If you're using a framework or the Claude Agent SDK, "spawn a sub-agent" gives you this isolation for free — a separate context window, its own tools, a summary returned. Understanding *why* it's structured this way is what lets you use it well.

## When isolation hurts

Isolation trades context cleanliness for coordination cost. It's the wrong call when:

- **Sub-tasks need to see each other's intermediate work.** If worker B genuinely needs worker A's raw output (not a summary), they aren't independent — reconsider the decomposition, or use handoffs.
- **The task is small.** Spawning an isolated agent to do something the orchestrator could do in two tool calls adds a round-trip and a summarization step for nothing.
- **The distillation loses something load-bearing.** If the orchestrator keeps needing detail the summary dropped, your `WorkerResult` schema is too lossy — widen it, or return a reference the orchestrator can expand on demand.

## Key takeaways

- Multi-agent value comes from **context economics**: sub-agents spend tokens in their own window and return little.
- Always return a **distilled result** (answer + sources + confidence + open questions), never the transcript.
- Seed each sub-agent with **only its assignment**, scope its tools, and force the return shape with a schema.
- Isolation costs a round-trip and a summarization — don't pay it for small or tightly-coupled work.
