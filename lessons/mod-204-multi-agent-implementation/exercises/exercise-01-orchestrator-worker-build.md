# exercise-01: Build an Orchestrator-Worker System

**Estimated effort:** 3 hours

## Objective

Build a working orchestrator-worker system from scratch: an orchestrator agent that decomposes a research question into independent sub-tasks, fans them out to worker agents that run concurrently, and synthesizes their distilled results into a final answer. By the end you'll understand exactly where decomposition quality and fan-out bounds make or break the system.

## Background

This exercise covers material from:

- [Chapter 1 — The Orchestrator-Worker Pattern](../01-orchestrator-worker.md)
- [Chapter 4 — Sub-Agent Isolation and Distilled Returns](../04-subagent-isolation.md)

Use any model provider and the agent loop you built in [mod-201](../../mod-201-agent-fundamentals/README.md). A web-search or document-retrieval tool makes the example realistic, but you may stub it with a fixed corpus.

## Prerequisites

- An agent loop with tool calling (from mod-201).
- `async` support so workers run concurrently (`asyncio` in Python, `Promise.all` in Node).
- API key in an environment variable; small spend cap.

## Tasks

### 1. The decomposition step

- Write an orchestrator call that takes a research question (e.g., *"Compare the managed Kubernetes offerings of AWS, GCP, and Azure on cost, autoscaling, and GPU support"*) and returns **structured** assignments: a list of `{worker_role, instruction}`.
- Force the structure with a schema / response model — do not parse prose.
- Cap the list at **N = 5** assignments and make the orchestrator respect the cap.

### 2. Concurrent fan-out

- Implement `run_workers(assignments)` that runs each worker **concurrently**, each as an isolated agent loop seeded with only its instruction.
- Each worker returns a distilled `WorkerResult` (answer + sources + confidence), not its transcript.

### 3. Synthesis

- Feed the workers' results back to the orchestrator with an explicit synthesis instruction: combine the findings, surface disagreements between workers, and do not introduce facts not present in the results.

### 4. Partial failure

- Make one worker fail deliberately. Show the system still completes: failed assignments are reported to the orchestrator, which synthesizes from the successes and notes what's missing.

### 5. Instrumentation

- Log, per run: number of assignments, per-worker token usage, total wall-clock, and which workers failed. You'll reuse this in [mod-205](../../mod-205-evaluation-observability/README.md).

## Starter guidance

```python
import asyncio
from pydantic import BaseModel

class Assignment(BaseModel):
    worker_role: str
    instruction: str

class WorkerResult(BaseModel):
    answer: str
    sources: list[str]
    confidence: str

async def decompose(question: str, max_n: int = 5) -> list[Assignment]:
    raise NotImplementedError  # orchestrator call returning structured assignments

async def run_worker(a: Assignment) -> WorkerResult:
    raise NotImplementedError  # isolated agent loop, distilled return

async def synthesize(question, results, failed) -> str:
    raise NotImplementedError

async def main(question: str) -> str:
    assignments = await decompose(question)
    settled = await asyncio.gather(*(run_worker(a) for a in assignments),
                                   return_exceptions=True)
    results = [r for r in settled if isinstance(r, WorkerResult)]
    failed  = [a for a, r in zip(assignments, settled) if not isinstance(r, WorkerResult)]
    return await synthesize(question, results, failed)
```

You do **not** need handoffs (exercise-02) or a real evaluator loop (exercise-04) here.

## Acceptance criteria

You can demonstrate that:

- The orchestrator returns **structured** assignments and never exceeds the cap of 5.
- Workers run **concurrently** (total wall-clock is near the slowest worker, not the sum).
- Each worker returns a distilled result; the orchestrator's context never contains a worker's raw tool output.
- A deliberately failing worker does **not** crash the run; the final answer notes the gap.
- Logs show per-worker token usage and the failure.

## Reflection

In `NOTES.md`:

1. Give a question where decomposition produced **overlapping** assignments. How did you tighten the orchestrator prompt to fix it?
2. Estimate the orchestrator's context size *with* distilled returns vs. if workers returned full transcripts.
3. When would a single agent have been the better choice for your question?

## Stretch goals

- Add a second round: after synthesis, let the orchestrator spawn bounded *follow-up* assignments to fill gaps.
- Give worker roles different system prompts and tools (a `numbers` worker with a calculator, a `prose` worker without) and route by role.
- Replace `asyncio.gather` with a bounded worker pool (max 3 concurrent) and measure the latency/cost trade-off.
