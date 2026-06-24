# Chapter 3 — LLM-as-Judge Scoring

Many agent outputs have no exact reference answer. "Is this synthesis faithful to the sources?" "Is this support reply helpful and on-policy?" "Did the agent answer the actual question?" There's no string to match against. The pragmatic tool here is **LLM-as-judge**: use a model, with a rubric, to score the output. It scales human-like judgment across thousands of runs cheaply — but a naive judge is wrong in systematic, dangerous ways, and the job is to build one whose scores you can trust.

## When to reach for a judge

Use a judge when the quality you care about is **judgeable but not checkable**: faithfulness, relevance, tone, instruction-following, coherence. Do **not** use a judge when a deterministic check would do — if you can assert exact match, validate a schema, run a test, or check a number, do that. A judge is slower, costs a model call, and is itself fallible. Reserve it for the genuinely fuzzy criteria.

## Rubric design is the whole game

A vague prompt ("rate this 1–10") produces noise. A good judge prompt is specific and structured:

- **Score on one dimension at a time.** A single "overall quality" score smears faithfulness, helpfulness, and tone together and is uninterpretable. Score each separately.
- **Prefer a small discrete scale or binary** over 1–10. Models can't reliably distinguish a 6 from a 7, but they can tell "faithful" from "not faithful." Binary or 1–3 scales calibrate far better.
- **Give concrete criteria and anchors.** Define what each score *means* with examples, so "pass" has the same meaning every run.
- **Demand a rationale before the score.** Forcing the judge to explain first (chain-of-thought) improves accuracy and gives you something to audit.
- **Return structured output**, so you can aggregate.

```python
class Judgment(BaseModel):
    reasoning: str                 # written BEFORE the verdict
    faithful: bool                 # one dimension, binary
    unsupported_claims: list[str]  # evidence for the verdict
```

## Pairwise beats absolute

Asking a judge "is A better than B?" is more reliable than "score A from 1–10." Models are better at **relative** comparison than absolute calibration. For comparing two prompts, two models, or a change against baseline, prefer **pairwise** judging — and when you do, randomize which candidate is "A" to defuse position bias (below).

## The biases that wreck a naive judge

A judge is a model, so it inherits model biases. Know them and defend:

- **Position bias** — judges favor the first (or last) option presented. Defense: randomize order, or run both orders and require agreement.
- **Verbosity bias** — judges rate longer answers higher regardless of quality. Defense: instruct explicitly that length is not quality; watch for it in calibration.
- **Self-preference** — a judge tends to favor outputs from its own model family. Defense: use a different model as judge than the one being evaluated when you can.
- **Sycophancy / leniency** — judges drift toward generous scores. Defense: adversarial framing ("find what's wrong") and a forced rationale.

## Calibration: trust, but verify

A judge you haven't validated is a random number generator with good PR. **Calibrate it against human labels:** hand-label a few dozen examples, run the judge on the same set, and measure agreement (exact agreement, or Cohen's kappa for chance-corrected agreement). If the judge agrees with humans ~85%+, you can trust it for that task; if it agrees 60%, your rubric is broken — fix the rubric, not the data. Re-calibrate when you change the rubric, the judge model, or the task.

```python
def agreement(judge_labels, human_labels) -> float:
    matches = sum(j == h for j, h in zip(judge_labels, human_labels))
    return matches / len(human_labels)
```

## Key takeaways

- Use a judge for **judgeable-but-not-checkable** quality; use a deterministic check whenever one exists.
- Design rubrics that score **one dimension at a time**, on a **small/binary scale**, with **concrete anchors** and a **rationale-before-verdict**, returning structured output.
- Prefer **pairwise** comparison over absolute scoring; it's more reliable.
- Defend against **position, verbosity, self-preference, and leniency** biases — and **calibrate against human labels** before you trust any judge.
