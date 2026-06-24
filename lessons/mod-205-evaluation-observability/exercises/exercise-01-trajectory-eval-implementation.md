# exercise-01: Trajectory Eval Implementation

**Estimated effort:** 3 hours

## Objective

Instrument an agent to capture its full trajectory, then build a trajectory evaluator that scores tool-call correctness, tool-call order, and efficiency against expected behavior — plus reference-free invariants that catch loops and bad arguments. By the end you'll be able to say not just "the answer was right" but "the agent took the right path to get there."

## Background

This exercise covers material from:

- [Chapter 1 — Trajectory and Tool-Call Evaluation](../01-trajectory-tool-call-eval.md)

Reuse the agent loop from [mod-201](../../mod-201-agent-fundamentals/README.md), and the per-run instrumentation you started in [mod-204 exercise-01](../../mod-204-multi-agent-implementation/exercises/exercise-01-orchestrator-worker-build.md). Any model provider works. Give the agent 2–3 tools (e.g., `search`, `calculator`, `lookup`) so trajectories have real branching.

## Prerequisites

- An agent loop with tool calling (from mod-201).
- 2–3 tools the agent can call; they can be stubs with deterministic outputs.
- API key in an environment variable; small spend cap.

## Tasks

### 1. Capture the trajectory

- Instrument the agent loop to append a structured `Step` on every iteration: reasoning turns, tool calls (name + args), tool results, and the final answer.
- Emit a `Trajectory` object per task. Do **not** reconstruct it from logs after the fact — capture it inside the loop.

### 2. Outcome eval

- Implement a final-answer check: exact match or assertion against a reference for at least 5 tasks where you know the answer.

### 3. Trajectory eval

- Implement **tool-call correctness**: each expected tool was called with valid arguments.
- Implement **tool-call order** using **in-order subset** matching (every expected call appears in the right relative order; benign extra steps allowed).
- Implement an **efficiency** metric: step count, and a flag for any identical tool call repeated 3+ times.

### 4. Reference-free invariants

- Assert no tool call had malformed/missing arguments.
- Assert the agent stopped within a step budget.
- Assert every source cited in the final answer actually appears in a tool result (catch hallucinated citations).

### 5. Run a small dataset

- Build 5–8 tasks, each with an expected tool set/order. Run all and print a per-task and aggregate report (outcome pass rate, trajectory match rate, mean steps).

## Starter guidance

```python
from dataclasses import dataclass, field

@dataclass
class Step:
    kind: str                       # "reason" | "tool_call" | "tool_result" | "answer"
    tool: str | None = None
    args: dict = field(default_factory=dict)
    output: str | None = None

@dataclass
class Trajectory:
    task: str
    steps: list[Step] = field(default_factory=list)
    final: str | None = None

def tool_calls(t: Trajectory) -> list[str]:
    return [s.tool for s in t.steps if s.kind == "tool_call"]

def in_order_subset(actual: list[str], expected: list[str]) -> bool:
    it = iter(actual)
    return all(tool in it for tool in expected)

def eval_trajectory(t: Trajectory, expected_tools: list[str], step_budget: int) -> dict:
    raise NotImplementedError  # correctness + order + efficiency + invariants
```

You do **not** need OTel (exercise-02) or a judge (exercise-03) here — keep checks deterministic.

## Acceptance criteria

You can demonstrate that:

- Every run produces a structured `Trajectory` captured *inside* the loop.
- Outcome eval and trajectory eval are scored **separately** for each task.
- In-order-subset matching passes when the agent takes a valid longer path and fails when it skips or reorders a required tool.
- A task you deliberately break (wrong tool, or a forced 3× repeat) is caught by the trajectory eval or an invariant, even if the final answer is correct.
- The report shows per-task and aggregate metrics.

## Reflection

In `NOTES.md`:

1. Give a task where the **answer was right but the trajectory was wrong**. What was the latent bug?
2. Which match mode (exact / in-order subset / set) fit your tasks best, and why did the stricter modes produce false failures?
3. Which reference-free invariant gave you the most signal for the least effort?

## Stretch goals

- Add single-step **precision/recall** for tool calls, aggregated across the dataset.
- Make the agent loop emit each step as it happens so you can watch the trajectory live.
- Feed the captured trajectory into the OTel spans you'll build in [exercise-02](exercise-02-otel-tracing-wireup.md) — same data, two representations.
