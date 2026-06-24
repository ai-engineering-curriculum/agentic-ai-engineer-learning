# exercise-03: LLM-as-Judge Scoring

**Estimated effort:** 2 hours

## Objective

Build an LLM-as-judge that scores agent outputs against a rubric, then calibrate it against your own human labels and measure agreement. By the end you'll have a judge whose scores you can actually trust — and a concrete number for how much you can trust them.

## Background

This exercise covers material from:

- [Chapter 3 — LLM-as-Judge Scoring](../03-llm-as-judge.md)

Score the outputs from a real agent task — the synthesis output from [mod-204](../../mod-204-multi-agent-implementation/README.md) or the answers from exercise-01 work well. Pick a fuzzy quality dimension that has **no exact reference** (faithfulness to sources, or relevance to the question).

## Prerequisites

- ~20–30 agent outputs to score, with their inputs/sources.
- A judge model (ideally a different model family than the one that produced the outputs).
- API key in an environment variable; small spend cap.

## Tasks

### 1. Hand-label first

- Before writing the judge, **you** label all 20–30 outputs on the chosen dimension using a binary or 1–3 scale. These are your ground truth.
- Write down your labeling criteria — you'll reuse them in the judge prompt.

### 2. Build the judge

- Write a judge prompt that scores **one dimension**, on a **binary or small** scale, with **concrete anchors**.
- Force a **rationale before the verdict** and return **structured output**.
- Include evidence in the output (e.g., a list of unsupported claims) so each verdict is auditable.

### 3. Calibrate

- Run the judge over the same set you hand-labeled.
- Compute agreement (exact agreement; optionally Cohen's kappa). Report the number.
- If agreement is low, **fix the rubric** (sharpen anchors, split the dimension), not the labels. Re-run and re-measure.

### 4. Probe for bias

- Run a **verbosity** check: does the judge favor longer outputs? Pad a known-good short answer and see if the score moves.
- If doing pairwise: swap A/B order and check the verdict is stable (position bias).

## Starter guidance

```python
from pydantic import BaseModel

class Judgment(BaseModel):
    reasoning: str                 # written BEFORE the verdict
    faithful: bool                 # one dimension, binary
    unsupported_claims: list[str]

JUDGE_PROMPT = """You are a strict evaluator. Judge ONLY whether every claim in the
ANSWER is supported by the SOURCES. Reason first, then decide. Length is not quality.

SOURCES:
{sources}

ANSWER:
{answer}

Return reasoning, then faithful (true/false), then any unsupported claims."""

def judge(answer: str, sources: str) -> Judgment:
    raise NotImplementedError  # structured-output call with JUDGE_PROMPT

def agreement(judge_labels, human_labels) -> float:
    matches = sum(j == h for j, h in zip(judge_labels, human_labels))
    return matches / len(human_labels)
```

Reach for a judge only because this dimension has no deterministic check — note where a check *would* have been better.

## Acceptance criteria

You can demonstrate that:

- You hand-labeled the set **before** building the judge.
- The judge scores one dimension on a binary/small scale, reasons before verdict, and returns structured, auditable output.
- You report a concrete **agreement number** against your human labels.
- Sharpening the rubric measurably **changed** agreement (you show before/after).
- You ran at least one **bias probe** (verbosity or position) and report what you found.

## Reflection

In `NOTES.md`:

1. What was your judge–human agreement, and would you trust this judge in a regression suite? Why or why not?
2. What rubric change moved agreement the most?
3. Which bias did you find, and how would you defend against it in production?

## Stretch goals

- Convert to **pairwise** judging (A vs. B) with order randomization and compare reliability to absolute scoring.
- Use a second judge model and only trust verdicts where the two judges agree; measure how much coverage you lose.
- Wire the judge into the regression suite from [exercise-04](exercise-04-agent-regression-suite.md) as a scored metric.
