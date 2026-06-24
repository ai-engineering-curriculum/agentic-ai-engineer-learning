# Chapter 4 — Regression Eval Suites

A one-off eval tells you how the agent does *today*. A **regression suite** tells you whether a change made it *worse* — and runs automatically, on every change, so a bad prompt edit or a model upgrade can't ship a silent regression. This is the piece that turns the techniques from Chapters 1–3 (trajectory checks, traces, judges) into an operational safety net. The model here is the unit test suite, lifted to non-deterministic agents.

```text
   PR / change ─▶ run eval suite over dataset ─▶ aggregate scores
                                                      │
                          ┌───────────────────────────┴──────────┐
                          ▼                                       ▼
                  scores ≥ thresholds                     scores < thresholds
                     → pass, merge                          → fail, block
```

## The dataset is the asset

A regression suite is only as good as its dataset. Build it deliberately:

- **Start from real failures.** Every production bug becomes a permanent test case. This is how the suite gets sharp over time — it accumulates exactly the cases that have actually broken.
- **Cover the task distribution**, not just the happy path: edge cases, adversarial inputs, the tool-error path, the empty-result path.
- **Pin each case to a checkable expectation** — a reference answer, an expected tool trajectory ([Chapter 1](01-trajectory-tool-call-eval.md)), or a judge rubric ([Chapter 3](03-llm-as-judge.md)).
- **Version the dataset** alongside code. An eval result is meaningless without knowing which dataset version produced it.

A few dozen well-chosen cases beat thousands of redundant ones. Coverage of failure modes matters more than raw count.

## Scoring: aggregate, with thresholds

Each case yields one or more scores; the suite aggregates them into metrics you gate on — pass rate, mean faithfulness, tool-call accuracy, p95 latency, cost per task. Set a **threshold** per metric and fail the run if any metric falls below it.

Because agents are non-deterministic, a single run is noisy. Two defenses:

- **Run each case `k` times** and aggregate (mean, or pass@k / majority), so one unlucky sample doesn't flip the gate.
- **Set thresholds with margin.** Don't gate at "≥ today's exact score" — a tiny stochastic dip would block every PR. Gate at a level that catches real regressions while tolerating noise (e.g., "pass rate ≥ 0.90" when today's is 0.94).

```python
def gate(results: dict[str, float], thresholds: dict[str, float]) -> bool:
    failures = {m: (results[m], t) for m, t in thresholds.items() if results[m] < t}
    for metric, (got, want) in failures.items():
        print(f"REGRESSION: {metric} = {got:.3f} < {want:.3f}")
    return not failures
```

## Gating changes in CI

Wire the suite into CI so it runs on every PR that touches prompts, tools, or model config. The flow: run the suite, compare against thresholds (or against the baseline branch's scores), block the merge on regression, and surface the per-case diff so the author sees *which* cases regressed and how. The cost discipline from [Chapter 2's tracing](02-otel-genai-tracing.md) matters here — a 500-case suite run `k` times per PR adds up, so size the suite and `k` against your budget.

## Beyond pass/fail: track drift

Even when the gate passes, log every run's metrics to a dashboard. Slow degradation — faithfulness drifting down a point a week as the dataset or a dependency shifts — won't trip a threshold but will quietly rot the system. A trend line catches what a single gate misses. This is the same observability discipline as tracing, applied to eval scores over time.

## Key takeaways

- A regression suite runs evals **automatically on every change** and **blocks regressions** before they ship — unit tests for a non-deterministic system.
- The **dataset is the asset**: grow it from real failures, cover failure modes, pin each case to a checkable expectation, and version it.
- Handle non-determinism with **`k` runs per case** and **thresholds set with margin**, gating on aggregate metrics.
- **Gate in CI** on PRs that touch prompts/tools/models, and **track metric drift** on a dashboard to catch slow degradation the gate won't.
