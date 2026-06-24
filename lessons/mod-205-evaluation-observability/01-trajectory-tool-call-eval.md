# Chapter 1 — Trajectory and Tool-Call Evaluation

Evaluating an agent is not the same as evaluating a model. A model eval asks: given this input, is the output right? An agent eval has to ask a harder question: across a *sequence* of decisions — which tool to call, with what arguments, in what order, when to stop — did the agent behave well? The final answer can be correct by luck while the path that produced it is broken, or the path can be flawless while the answer is wrong because of a bad tool result. You need to grade both.

```text
   task ─▶ [reason] ─▶ tool_call(search, q="...") ─▶ tool_result
              │                                          │
              ▼                                          ▼
          [reason] ─▶ tool_call(calc, expr="...") ─▶ tool_result ─▶ [answer]

   the trajectory = this whole ordered sequence of steps
```

## What a trajectory is

A **trajectory** is the ordered record of everything the agent did for one task: each reasoning turn, each tool call (name + arguments), each tool result, and the final output. To evaluate it, you first have to *capture* it. Instrument your agent loop to emit a structured step on every iteration — don't try to reconstruct it from logs after the fact.

```python
from dataclasses import dataclass, field

@dataclass
class Step:
    kind: str            # "reason" | "tool_call" | "tool_result" | "answer"
    tool: str | None = None
    args: dict = field(default_factory=dict)
    output: str | None = None

@dataclass
class Trajectory:
    task: str
    steps: list[Step] = field(default_factory=list)
    final: str | None = None
```

## Two layers of evaluation

**Outcome (final-answer) eval** is the familiar one: compare `trajectory.final` to a reference with exact match, an assertion, or a judge ([Chapter 3](03-llm-as-judge.md)). Necessary, not sufficient.

**Trajectory (process) eval** grades the path. The most useful checks:

- **Tool-call correctness** — did the agent call the *right* tool with valid arguments? A weather query that calls `calculator` is wrong even if the final answer happens to be right.
- **Tool-call order** — for tasks with a required sequence (authenticate → fetch → format), is the order respected? Compare against an expected ordered set, allowing for benign extra steps.
- **Efficiency** — did it reach the answer in a reasonable number of steps, or loop? Step count and repeated identical calls are cheap, high-signal metrics.
- **No forbidden actions** — did it avoid tools it shouldn't touch for this task (a read-only query that issued a write)?

## Match modes, from strict to lenient

Real agents take different valid paths to the same answer, so a rigid "trajectory must equal reference exactly" match produces constant false failures. Pick the loosest mode that still catches real bugs:

- **Exact match** — the step sequence equals the reference exactly. Brittle; use only for tightly scripted flows.
- **In-order subset** — every expected tool call appears in the right relative order; extra steps are allowed. The pragmatic default.
- **Set match** — the *set* of tools used equals the expected set; order ignored. Use when order genuinely doesn't matter.
- **Single-step precision/recall** — score how many tool calls were appropriate vs. how many appropriate calls were missed, aggregated across the dataset.

```python
def in_order_subset(actual: list[str], expected: list[str]) -> bool:
    it = iter(actual)
    return all(tool in it for tool in expected)   # each expected found, in order
```

## Reference-free trajectory checks

You won't always have a hand-written reference trajectory — they're expensive. Many high-value checks need no reference at all: assert no tool call had malformed arguments, assert the agent stopped within a step budget, assert it never repeated the identical call three times in a row, assert every cited source actually appears in a tool result. These reference-free invariants catch loops, argument bugs, and hallucinated citations cheaply, and they scale to any task.

## Key takeaways

- An agent eval grades the **trajectory** (the ordered step sequence), not only the final answer — a right answer via a wrong path is a latent bug.
- Capture trajectories by **instrumenting the loop** to emit structured steps; don't reconstruct from logs.
- Score two layers: **outcome** (final answer) and **process** (tool-call correctness, order, efficiency, forbidden actions).
- Use the **loosest match mode** that still catches bugs — in-order subset is the pragmatic default — and lean on **reference-free invariants** for cheap, scalable coverage.
