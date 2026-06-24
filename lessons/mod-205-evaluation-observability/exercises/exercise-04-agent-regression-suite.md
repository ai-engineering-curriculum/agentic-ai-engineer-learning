# exercise-04: Agent Regression Suite

**Estimated effort:** 2 hours

## Objective

Assemble a dataset-backed regression suite that scores your agent across trajectory, outcome, and judge metrics, gates on thresholds, and blocks a regression in CI. By the end you'll have a safety net that catches a bad prompt or model change before it ships — the operational payoff of everything in this module.

## Background

This exercise covers material from:

- [Chapter 4 — Regression Eval Suites](../04-regression-eval-suites.md)

This exercise composes the previous three: the trajectory eval from [exercise-01](exercise-01-trajectory-eval-implementation.md), and optionally the judge from [exercise-03](exercise-03-llm-judge-scoring.md), become the scorers your suite runs over a dataset.

## Prerequisites

- The trajectory evaluator from exercise-01.
- `pytest` (or any runner) and a CI environment (GitHub Actions is fine).
- API key in CI secrets; a spend cap sized for repeated runs.

## Tasks

### 1. Build and version the dataset

- Assemble 8–15 cases covering the task distribution: happy path, an edge case, an adversarial input, a tool-error path, an empty-result path.
- Pin each case to a checkable expectation: a reference answer, an expected tool trajectory, and/or a judge rubric.
- Store the dataset as a versioned file (JSON/YAML) committed alongside code.

### 2. Wire the scorers

- For each case, run the agent and apply the relevant scorers (outcome check, trajectory match, optional judge).
- Aggregate into metrics: pass rate, tool-call accuracy, mean steps, and (if used) mean judge score.

### 3. Handle non-determinism

- Run each case `k` times (start with `k=3`) and aggregate so one unlucky sample doesn't flip the gate.

### 4. Gate on thresholds

- Define a threshold per metric, set **with margin** below today's score.
- Fail the run (non-zero exit) if any metric falls below its threshold, printing which metric regressed and by how much.

### 5. Run it in CI

- Add a CI job that runs the suite on PRs touching prompts/tools/model config and blocks the merge on regression.
- Demonstrate the gate: make a change that regresses the agent (e.g., a worse prompt) and show CI goes red; revert and show it goes green.

## Starter guidance

```python
import json

def load_dataset(path: str) -> list[dict]:
    with open(path) as f:
        return json.load(f)            # versioned cases with expectations

def run_case(case: dict, k: int = 3) -> dict:
    raise NotImplementedError          # run agent k times, apply scorers, aggregate

def aggregate(case_results: list[dict]) -> dict:
    raise NotImplementedError          # -> {"pass_rate": ..., "tool_acc": ..., "mean_steps": ...}

def gate(results: dict[str, float], thresholds: dict[str, float]) -> bool:
    failures = {m: (results[m], t) for m, t in thresholds.items() if results[m] < t}
    for metric, (got, want) in failures.items():
        print(f"REGRESSION: {metric} = {got:.3f} < {want:.3f}")
    return not failures

THRESHOLDS = {"pass_rate": 0.85, "tool_acc": 0.80}   # set below today's scores, with margin

if __name__ == "__main__":
    ds = load_dataset("dataset.json")
    results = aggregate([run_case(c) for c in ds])
    raise SystemExit(0 if gate(results, THRESHOLDS) else 1)
```

## Acceptance criteria

You can demonstrate that:

- The dataset is **versioned**, covers failure modes (not just the happy path), and pins each case to a checkable expectation.
- Each case runs `k` times and metrics are **aggregated**.
- Thresholds are set **with margin**; the gate fails loudly and names the regressed metric.
- A deliberately bad change makes **CI go red**, and reverting makes it green.
- No secrets are hardcoded — credentials come from CI secrets.

## Reflection

In `NOTES.md`:

1. Which failure-mode case in your dataset came from a real (or realistic) bug, and why is it worth keeping forever?
2. How did you choose `k` and your thresholds to balance noise tolerance against catching real regressions?
3. What's the cost of one full suite run, and how would you keep it affordable as the dataset grows?

## Stretch goals

- Compare against the **baseline branch's** scores instead of fixed thresholds, and report a per-case diff.
- Log every run's metrics to a dashboard and watch for **drift** across runs, not just pass/fail.
- Add the judge from exercise-03 as a gated metric, and calibrate the threshold against its measured agreement.
