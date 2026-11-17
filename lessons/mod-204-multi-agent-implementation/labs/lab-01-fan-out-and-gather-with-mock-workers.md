# Lab 1 — Fan Out and Gather with Mock Workers

**Duration:** ~45 minutes
**Covers:** [Chapter 1 — The Orchestrator-Worker Pattern](../01-orchestrator-worker.md)

## Objective

Before Exercise 01 asks you to wire an orchestrator-worker system against a real model, spend 45 minutes building the **control-flow skeleton** against *mock* workers. You'll see the three Chapter 1 phases — decompose, fan out concurrently, synthesize — plus bounded fan-out and partial-failure handling, without burning a single token.

By the end of the lab you will:

- Define `Assignment` and `WorkerResult` as pydantic models that match Chapter 1's shape.
- Fan out mock workers with `asyncio.gather` and watch wall-clock time stay near the **slowest** worker, not the sum.
- Handle a deliberately failing worker so the run still completes with a synthesized answer.

This lab is intentionally narrower than Exercise 01: no real API, no logging, no corpus, no tools. Just the plumbing.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 1](../01-orchestrator-worker.md) read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install pydantic
mkdir lab-01 && cd lab-01
```

## Step 1 — The two shapes that cross agent boundaries

Chapter 1 says the orchestrator should return *structured* assignments, and workers should return *distilled* results. Define both now, as pydantic models — so every later step is type-checked.

Save as `shapes.py`:

```python
from pydantic import BaseModel
from typing import Literal


class Assignment(BaseModel):
    worker_role: str
    instruction: str


class WorkerResult(BaseModel):
    answer: str
    sources: list[str]
    confidence: Literal["high", "medium", "low"]
```

Notice: the fields match Chapter 1 and Chapter 4 verbatim. We are not inventing a dialect.

## Step 2 — A mock decomposer that respects the cap

Chapter 1 insists on a **bounded** fan-out — the orchestrator must emit at most N assignments. Simulate it with a hand-written list and a guardrail.

Save as `decompose.py`:

```python
from shapes import Assignment

MAX_ASSIGNMENTS = 5


def decompose(question: str) -> list[Assignment]:
    """Mock orchestrator decomposition.

    In the real exercise this is a model call constrained by a response schema.
    Here we hand-write the assignments so we can focus on the fan-out plumbing.
    """
    raw = [
        {"worker_role": "profile", "instruction": f"Profile AWS re: {question}"},
        {"worker_role": "profile", "instruction": f"Profile GCP re: {question}"},
        {"worker_role": "profile", "instruction": f"Profile Azure re: {question}"},
    ]
    assignments = [Assignment(**a) for a in raw]
    if len(assignments) > MAX_ASSIGNMENTS:
        raise ValueError(
            f"orchestrator emitted {len(assignments)} assignments; cap is {MAX_ASSIGNMENTS}"
        )
    return assignments
```

**Why a guardrail on a hand-written list:** it's muscle memory. Exercise 01's model-driven decomposer will occasionally emit 7 assignments despite your prompt saying "at most 5." The runtime, not the model, enforces the cap.

## Step 3 — Mock workers with knowable latency

Give each worker an `asyncio.sleep` so we can measure concurrency. One worker deliberately raises.

Save as `workers.py`:

```python
import asyncio
from shapes import Assignment, WorkerResult


async def run_worker(a: Assignment) -> WorkerResult:
    """Mock worker: sleeps 1 second, returns a distilled WorkerResult.

    Chapter 4: the worker's own tool calls and reasoning never cross back —
    we return only answer + sources + confidence.
    """
    await asyncio.sleep(1.0)

    if "Azure" in a.instruction:
        # Deliberate failure to exercise the partial-failure path.
        raise RuntimeError("mock worker failed: Azure profile source timed out")

    return WorkerResult(
        answer=f"Mock finding for: {a.instruction}",
        sources=[f"https://example.test/{a.worker_role}/1"],
        confidence="medium",
    )
```

## Step 4 — Fan out with `asyncio.gather(return_exceptions=True)`

This is the heart of the pattern. `return_exceptions=True` is what makes partial failure a design choice instead of a crash.

Save as `fanout.py`:

```python
import asyncio
import time
from shapes import Assignment, WorkerResult
from workers import run_worker


async def fan_out(assignments: list[Assignment]) -> tuple[list[WorkerResult], list[Assignment]]:
    """Run every worker concurrently. Return (successes, failed_assignments)."""
    t0 = time.monotonic()
    settled = await asyncio.gather(
        *(run_worker(a) for a in assignments),
        return_exceptions=True,
    )
    elapsed = time.monotonic() - t0

    results, failed = [], []
    for a, r in zip(assignments, settled):
        if isinstance(r, WorkerResult):
            results.append(r)
        else:
            failed.append(a)

    print(f"fan-out: {len(assignments)} assignments, "
          f"{len(results)} ok, {len(failed)} failed, {elapsed:.2f}s wall-clock")
    return results, failed
```

**Why `return_exceptions=True` and not a `try/except` around `gather`:** without it, the *first* exception bubbles and the other workers' results are lost. Chapter 1's "handle partial failure" rule only works if you keep what came back.

## Step 5 — A mock synthesize that trusts only what workers returned

Save as `synthesize.py`:

```python
from shapes import Assignment, WorkerResult


def synthesize(question: str,
               results: list[WorkerResult],
               failed: list[Assignment]) -> str:
    """Mock synthesis: concatenate distilled answers + note what failed.

    In the real exercise this is a model call whose system prompt explicitly
    forbids introducing facts not present in the worker results.
    """
    lines = [f"Question: {question}", ""]
    for r in results:
        lines.append(f"- [{r.confidence}] {r.answer}  (sources: {', '.join(r.sources)})")
    if failed:
        lines.append("")
        lines.append("Missing coverage:")
        for a in failed:
            lines.append(f"- {a.worker_role}: {a.instruction}")
    return "\n".join(lines)
```

## Step 6 — Run it

Save as `main.py`:

```python
import asyncio
from decompose import decompose
from fanout import fan_out
from synthesize import synthesize


async def main() -> None:
    question = "Compare managed Kubernetes on cost, autoscaling, and GPUs"
    assignments = decompose(question)
    results, failed = await fan_out(assignments)
    final = synthesize(question, results, failed)
    print("=" * 60)
    print(final)


if __name__ == "__main__":
    asyncio.run(main())
```

Run it:

```bash
python main.py
```

**Expected output (shape, not exact bytes):**

```
fan-out: 3 assignments, 2 ok, 1 failed, 1.00s wall-clock
============================================================
Question: Compare managed Kubernetes on cost, autoscaling, and GPUs

- [medium] Mock finding for: Profile AWS re: Compare managed Kubernetes...  (sources: https://example.test/profile/1)
- [medium] Mock finding for: Profile GCP re: Compare managed Kubernetes...  (sources: https://example.test/profile/1)

Missing coverage:
- profile: Profile Azure re: Compare managed Kubernetes on cost, autoscaling, and GPUs
```

Two things to notice:

- **Wall-clock is ~1.0s, not ~3.0s.** Three one-second workers ran concurrently. If you saw 3.0s you accidentally `await`ed them in a loop instead of inside `asyncio.gather`.
- **The failing Azure worker did not sink the run.** The synthesizer noted the gap and produced a best-effort answer from what came back.

## Step 7 — Break the fan-out bound on purpose

In `decompose.py`, add three more hand-written assignments so the list grows to 6. Rerun `python main.py`. You should see a `ValueError: orchestrator emitted 6 assignments; cap is 5` — and the run never reaches the fan-out. That's the bound the runtime (not the model) enforces.

Put it back to 3 when done.

## Step 8 — Replace `gather` with a serial loop and watch the latency change

Temporarily replace the body of `fan_out` with:

```python
    t0 = time.monotonic()
    settled = []
    for a in assignments:
        try:
            settled.append(await run_worker(a))
        except Exception as e:
            settled.append(e)
    elapsed = time.monotonic() - t0
```

Rerun. Wall-clock should now be ~3.0s (one worker at a time). This is the latency price of *not* fanning out — and the cost Chapter 1 is paying to avoid when it says "run the workers **concurrently**."

Restore `gather` when done.

**Checkpoint:** you can state in your own words:

1. Why `asyncio.gather(..., return_exceptions=True)` is the right primitive for a fan-out that must survive a bad worker.
2. Which piece of your code (the runtime, not the prompt) enforces the `MAX_ASSIGNMENTS` cap.
3. What distinguishes a `WorkerResult` from "the worker's transcript" — and why only the former crosses back to the synthesizer.

## Wrap-up

Carry the three mock files into Exercise 01 as a skeleton. Replace `decompose.py` with a real orchestrator call constrained by the `Assignment` response schema; replace `workers.py` with an isolated agent loop (seeded with only `a.instruction`, forced into the `WorkerResult` schema); replace `synthesize.py` with a model call carrying the "do not introduce facts not present in the results" instruction. The orchestration plumbing — `gather`, `return_exceptions`, the `MAX_ASSIGNMENTS` cap, the failed-assignment reporting — is already the shape Exercise 01 asks for.
