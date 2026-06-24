# exercise-05: Framework Tradeoff Bakeoff

**Estimated effort:** 2 hours

## Objective

Take your three implementations of the identical research-assistant task — LangGraph (exercise 01), CrewAI (exercise 02), and AutoGen (exercise 03) — run them through one shared harness, and produce a side-by-side scorecard plus a one-page decision memo. This is where the qualitative rankings from Chapter 5 become real numbers for *your* task, and where you practice the SDK-vs-framework judgment from Chapter 6.

## Background

This exercise covers material from:

- [Chapter 5 — Comparing LangGraph, CrewAI, and AutoGen](../05-comparing-langgraph-crewai-autogen.md)
- [Chapter 6 — Evaluating Competing SDKs](../06-evaluating-competing-sdks.md)

You are not writing a new agent here. You are measuring the three you already built against a common rubric.

## Prerequisites

- Completed [exercise-01](exercise-01-langgraph-stateful-agent.md), [exercise-02](exercise-02-crewai-role-based-crew.md), and [exercise-03](exercise-03-autogen-multi-agent.md).
- The same `search`/`calc` tools and the same demo task across all three (ideally the MCP server from [exercise-04](exercise-04-mcp-tool-server.md), so the tool layer is identical).

## Tasks

### 1. Build the shared harness

- Write `run_build(name, invoke) -> Metrics` that runs one framework build on the fixed task and records: total LLM calls, total tokens (prompt + completion), wall-clock latency, and whether the final answer is correct against a known expected ratio.
- Run every build the **same** number of times (≥ 3) and report medians to dampen variance.

### 2. Score on the shared rubric

- Score each build on the axes from Chapter 5: mental model fit, lines of code, control-flow clarity, ease of capping cost, checkpointing/resumability, debuggability, and observability hookup.
- Use a fixed scale (e.g. 1–5) and write a one-line justification per cell — no bare numbers.

### 3. Capture cost and latency

- Tabulate the measured metrics from task 1 next to the qualitative scores. Note where the measurement contradicts your prior expectation from the chapter.

### 4. Make the SDK-vs-framework call

- For the *same* task, sketch (don't fully build) what an SDK-only version (OpenAI Agents SDK, Claude Agent SDK, or smolagents) would look like. Estimate its line count and decide whether the framework earned its weight on this task.

### 5. Write the decision memo

- One page: given a concrete (real or invented) product requirement, recommend one framework or SDK, cite the scorecard rows that drove the decision, and name what would change your mind.

## Starter guidance

```python
from dataclasses import dataclass
from typing import Callable


@dataclass
class Metrics:
    name: str
    llm_calls: int
    total_tokens: int
    latency_s: float
    correct: bool


EXPECTED_RATIO = 1.0  # set to the known answer for your fixed task; tolerance-checked


def run_build(name: str, invoke: Callable[[str], str], task: str, trials: int = 3) -> Metrics:
    raise NotImplementedError  # run `trials` times, capture token usage + wall clock, median


def score_rubric(name: str) -> dict[str, tuple[int, str]]:
    raise NotImplementedError  # {axis: (1..5, "one-line justification")}


if __name__ == "__main__":
    task = ("Compare the population of the two most populous countries in 2024 "
            "and report the ratio.")
    builds = {
        "langgraph": lambda t: ...,   # wrap exercise-01 graph.invoke
        "crewai":    lambda t: ...,   # wrap exercise-02 crew.kickoff
        "autogen":   lambda t: ...,   # wrap exercise-03 initiate_chat
    }
    results = [run_build(name, fn, task) for name, fn in builds.items()]
    for m in results:
        print(m)
```

## Acceptance criteria

You can demonstrate that:

- All three builds run through one harness on the identical task with the identical tool layer.
- The scorecard reports measured LLM calls, tokens, latency (medians over ≥ 3 trials), and correctness per build.
- The rubric scores each build on every Chapter 5 axis with a one-line justification per cell.
- At least one measured result either confirms or contradicts a Chapter 5 expectation, and you say which.
- The decision memo recommends one option, cites specific scorecard rows, and states what would reverse the recommendation.

## Reflection

In `NOTES.md`:

1. Which framework had the lowest token spend on your task, and did that match the Chapter 5 prediction? If not, why?
2. Where did lines-of-code mislead you — was the shortest build actually the easiest to operate?
3. For your invented product requirement, would an SDK-only build have been enough? What single requirement tips it back to a framework?

## Stretch goals

- Add a fourth column for an SDK-only build (actually implement the sketch from task 4) and fold it into the scorecard.
- Vary the task difficulty (a 2-step vs. a 6-step question) and show how the rankings shift with complexity.
- Add an observability hook (LangSmith, Langfuse, or OpenTelemetry) to each build and compare how much wiring each one needed.
- Re-run the harness on a cheaper model and report how the cost/quality frontier moves per framework.
