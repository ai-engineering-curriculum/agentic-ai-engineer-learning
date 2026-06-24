# Quiz 1 — Evaluation & Observability Instrumentation

Knowledge check for [mod-205](../README.md). Answers are at the bottom; try each question before scrolling. Covers all four chapters.

## Questions

### 1. Trajectory vs. outcome

An agent answers a weather question correctly but reaches the answer by calling the `calculator` tool instead of the `weather` tool. Under a two-layer agent eval, what happens?

- A. Both outcome and trajectory eval pass — the answer was right.
- B. Outcome eval passes; trajectory eval fails on tool-call correctness.
- C. Both fail, because the tool was wrong.
- D. Neither layer can detect this.

### 2. Match modes

Your agents reach the same correct answer through different valid tool orderings, and a strict "trajectory must equal reference exactly" check keeps failing good runs. Which match mode is the pragmatic default?

- A. Exact match.
- B. In-order subset.
- C. Single-step recall only.
- D. No match mode — disable trajectory eval.

### 3. Reference-free checks

Which of these requires **no** hand-written reference trajectory?

- A. Comparing the step sequence to a gold path exactly.
- B. Asserting the agent stopped within a step budget.
- C. Checking the final answer against a reference string.
- D. Pairwise comparison against a baseline trajectory.

### 4. GenAI conventions

Why instrument spans with the OTel `gen_ai.*` semantic conventions instead of inventing your own attribute keys?

- A. Custom keys are forbidden by the OTLP protocol.
- B. Standard keys let platforms compute cost, latency, and token charts automatically.
- C. The `gen_ai.*` keys are faster to serialize.
- D. It has no practical effect; it's a style preference.

### 5. Span naming

What is the conventional span name for a single model chat call under the GenAI conventions?

- A. `model_call`
- B. `gen_ai.chat`
- C. `chat {model}`
- D. `llm.invoke`

### 6. Judge scales

Why is a binary or 1–3 judge scale usually preferable to a 1–10 scale?

- A. It costs fewer tokens.
- B. Models can't reliably distinguish adjacent points on a wide scale, so small scales calibrate better.
- C. 1–10 scales are not supported by structured output.
- D. Binary scales eliminate all judge bias.

### 7. Judge bias

A judge consistently rates longer answers higher regardless of quality. What is this, and one defense?

- A. Position bias; randomize order.
- B. Verbosity bias; instruct that length isn't quality and watch for it in calibration.
- C. Self-preference; use a different judge model.
- D. Sycophancy; use adversarial framing.

### 8. Calibration

You build a judge and it agrees with your human labels only 60% of the time. What should you do?

- A. Trust it anyway — 60% beats nothing.
- B. Relabel the human examples to match the judge.
- C. Fix the rubric (sharpen anchors, split the dimension) and re-measure.
- D. Switch to exact-match scoring for this fuzzy dimension.

### 9. Regression thresholds

Today your suite's pass rate is 0.94. Why gate at "pass rate ≥ 0.90" rather than "≥ 0.94"?

- A. To make the suite run faster.
- B. Because agents are non-deterministic; a tiny stochastic dip at 0.94 would block every PR.
- C. Because 0.90 is the industry-standard threshold.
- D. There's no reason; gate at the exact current score.

### 10. The dataset

What is the single best source of new regression-suite test cases over time?

- A. Randomly generated inputs.
- B. Every production failure, turned into a permanent case.
- C. The largest public benchmark you can find.
- D. Duplicating happy-path cases for volume.

## Answer key

1. **B** — A right answer via the wrong tool path is a latent bug; outcome eval passes but trajectory eval catches it ([Chapter 1](../01-trajectory-tool-call-eval.md)).
2. **B** — In-order subset allows benign extra steps while enforcing required order; it's the pragmatic default.
3. **B** — A step-budget assertion is a reference-free invariant; the others all need a reference.
4. **B** — Standard `gen_ai.*` keys are what let platforms render cost/latency/token charts for free ([Chapter 2](../02-otel-genai-tracing.md)).
5. **C** — The convention is `chat {model}` for a model call (and `execute_tool {name}` for a tool call).
6. **B** — Models can't reliably tell a 6 from a 7; binary/small scales calibrate far better ([Chapter 3](../03-llm-as-judge.md)).
7. **B** — Verbosity bias; defend by instructing that length isn't quality and checking for it during calibration.
8. **C** — Low agreement means the rubric is broken; fix the rubric, not the data, then re-measure.
9. **B** — Non-determinism makes a single run noisy; set thresholds with margin so noise doesn't block good PRs ([Chapter 4](../04-regression-eval-suites.md)).
10. **B** — Turning every production failure into a permanent case is how the suite gets sharp over time.
