# exercise-04: RAG Triad Evaluation

**Estimated effort:** 3 hours

## Objective

Turn "it feels better" into a number. Build a RAG evaluation harness that scores the **RAG triad** — context relevance, groundedness, and answer relevance — over a fixed evaluation set, then use it to compare the retrievers you built in exercise-03 and *prove* which one wins. By the end you'll read the three scores together to localize a fault to retrieval, generation, or prompting.

## Background

This exercise covers material from:

- [Chapter 4 — Advanced Retrieval and Evaluation](../04-advanced-rag-eval.md)

You'll evaluate the pipelines from [exercise-01](exercise-01-rag-pipeline-vector-db.md) and [exercise-03](exercise-03-advanced-rag-retrieval.md). You may build the LLM-as-judge scorers by hand or use **RAGAS**; either way you must understand what each metric measures and why a low score points where it does.

## Prerequisites

- At least two RAG pipelines to compare (naive from exercise-01, plus sentence-window or auto-merging from exercise-03).
- An LLM for the judge, with a small spend cap. The judge can be the same model family as the generator — but note the bias risk.
- 15-25 evaluation questions over your corpus.

## Tasks

### 1. Build a frozen eval set

- Write 15-25 `(question, ground_truth_answer)` rows over your corpus, including a few questions whose answer is **not** in the corpus (the right answer is "I don't know").
- Freeze it. A score change must reflect a pipeline change, not a different test set.

### 2. Score context relevance

- For each question, judge whether the retrieved chunks are relevant to the question (e.g., fraction of retrieved chunks that are on-topic).
- This isolates **retrieval** quality — the generator never sees what retrieval missed.

### 3. Score groundedness

- For each answer, judge whether every claim is supported by the retrieved context (e.g., fraction of answer sentences traceable to a chunk).
- This catches **hallucination**: a claim with no support in the context.

### 4. Score answer relevance

- Judge whether the answer actually addresses the question, independent of whether it's grounded.

### 5. Compare and localize

- Run all three metrics over the frozen set for each pipeline. Produce a table: pipeline × metric.
- Use the **joint reading** from Chapter 4 to localize at least one fault: low context relevance → retrieval; high context relevance + low groundedness → generation; high both + low answer relevance → prompt.

## Starter guidance

```python
from dataclasses import dataclass

@dataclass
class EvalRow:
    question: str
    ground_truth: str

@dataclass
class TriadScore:
    context_relevance: float
    groundedness: float
    answer_relevance: float

def judge_context_relevance(question: str, contexts: list[str]) -> float:
    raise NotImplementedError  # LLM-as-judge: fraction of contexts on-topic

def judge_groundedness(answer: str, contexts: list[str]) -> float:
    raise NotImplementedError  # fraction of answer claims supported by contexts

def judge_answer_relevance(question: str, answer: str) -> float:
    raise NotImplementedError  # does the answer address the question?

def evaluate(pipeline, eval_set: list[EvalRow]) -> dict[str, float]:
    raise NotImplementedError  # mean of each metric across the frozen set
```

## Acceptance criteria

You can demonstrate that:

- A frozen eval set of 15-25 rows exists, including out-of-corpus questions.
- All three triad metrics are computed per question and averaged per pipeline.
- A pipeline × metric table compares at least two pipelines (e.g., naive vs. an advanced retriever from exercise-03).
- You read the scores **together** to localize at least one concrete fault to retrieval, generation, or prompting — and explain the reasoning.
- The out-of-corpus questions are scored correctly: an "I don't know" answer is **not** penalized as ungrounded.

## Reflection

In `NOTES.md`:

1. Which advanced retriever from exercise-03 actually won on context relevance — and did your eyeballed comparison from that exercise agree with the numbers?
2. Find a case where groundedness and answer relevance disagree (grounded but off-topic, or relevant but unsupported). What does each combination tell you to fix?
3. How much do you trust the LLM judge? Hand-label 5 rows yourself and report where the judge disagreed with you.

## Stretch goals

- Add a **retrieval metric that needs ground truth** (context recall / precision against labeled relevant chunks) and compare it to the reference-free context-relevance judge.
- Sweep a parameter (chunk size, k, or re-rank top-k) and plot each triad metric against it — find the setting that maximizes the triad.
- Wire the harness into a CI check that fails the build if any triad metric drops below a threshold on the frozen set. You'll formalize this in [mod-205](../../mod-205-evaluation-observability/README.md).
