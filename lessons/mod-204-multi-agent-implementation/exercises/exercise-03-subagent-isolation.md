# exercise-03: Sub-Agent Isolation

**Estimated effort:** 3 hours

## Objective

Prove the context economics that make multi-agent systems worthwhile. You'll build an investigation task where a sub-agent does heavy work in its own context window and returns a small distilled result — then measure the orchestrator's context with and without isolation, so the savings are a number you've seen, not a claim you've read.

## Background

This exercise covers material from:

- [Chapter 4 — Sub-Agent Isolation and Distilled Returns](../04-subagent-isolation.md)
- [Chapter 1 — The Orchestrator-Worker Pattern](../01-orchestrator-worker.md)

## Prerequisites

- The agent loop from mod-201 with tool calling and token-usage reporting.
- A corpus the sub-agent can dig through: a directory of files, a docs set, or a stubbed search tool over ~20+ documents.

## Tasks

### 1. The heavy sub-agent

- Build a `researcher` sub-agent that, given a question, makes **many** tool calls over the corpus (read/search ≥ 10 times) before answering.
- Seed it with **only its assignment** — a fresh message list, never the parent's history.

### 2. Distilled return

- Force the sub-agent's output into a schema: `{answer, sources, confidence, open_questions}`. `sources` are references (ids/paths/urls), not file contents.
- Its system prompt must instruct: report the finding and supporting sources; **do not** include search process or raw tool output.

### 3. The orchestrator

- The orchestrator spawns the sub-agent, receives only the distilled result, and writes a final answer. The orchestrator must never see the sub-agent's tool calls.

### 4. Measure the asymmetry

- Instrument **both** windows. Report: (a) total tokens the sub-agent consumed internally, and (b) tokens the orchestrator received from the sub-agent.
- Then build a **non-isolated** variant: run the same investigation inline in the orchestrator (all tool results land in the orchestrator's context). Record its context size.
- Put the three numbers in a table.

### 5. Find the breaking point

- Increase the corpus / number of tool calls until the **non-isolated** version's context becomes a problem (degraded answers, or you hit a context limit). Note where the isolated version is still fine.

## Starter guidance

```python
from pydantic import BaseModel

class WorkerResult(BaseModel):
    answer: str
    sources: list[str]
    confidence: str
    open_questions: list[str]

async def run_isolated(assignment: str) -> WorkerResult:
    messages = [{"role": "user", "content": assignment}]   # fresh — no parent context
    return await agent_loop(system=RESEARCHER, messages=messages,
                            tools=CORPUS_TOOLS, response_model=WorkerResult)

async def run_inline(assignment: str, orchestrator_messages: list) -> str:
    # contrast: tool results accumulate in the orchestrator's own context
    raise NotImplementedError
```

## Acceptance criteria

You can demonstrate that:

- The sub-agent makes ≥ 10 tool calls yet the orchestrator receives only a distilled `WorkerResult`.
- The sub-agent's message list is seeded with **only** its assignment (show it).
- Your table reports sub-agent internal tokens, tokens crossing back, and the non-isolated context size — and the asymmetry is large.
- You can point to the corpus size at which the non-isolated version degrades while the isolated one holds.

## Reflection

In `NOTES.md`:

1. What was your ratio of sub-agent-internal tokens to tokens-returned? What does that ratio buy you?
2. Did your distilled schema ever drop something the orchestrator needed? How did you widen it without re-inlining the transcript?
3. Name a task where isolation would *hurt* — where the orchestrator genuinely needs the raw work.

## Stretch goals

- Add an `expand(source_id)` tool so the orchestrator can pull detail on demand for a specific source instead of receiving everything up front.
- Run three isolated sub-agents concurrently and confirm the orchestrator's context grows by ~3 small summaries, not 3 transcripts.
- Swap the distilled schema for "return the full transcript" and watch the orchestrator's answer quality and cost change — quantify it.
