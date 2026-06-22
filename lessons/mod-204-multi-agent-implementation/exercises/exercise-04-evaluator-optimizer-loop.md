# exercise-04: Evaluator-Optimizer Loop

**Estimated effort:** 3 hours

## Objective

Build a generator + critic loop that iterates a piece of work to a quality bar — and, just as importantly, that **stops**. You'll wire a structured verdict, a reachable pass condition, and a no-progress exit, then watch it both succeed and correctly give up.

## Background

This exercise covers material from:

- [Chapter 5 — The Evaluator-Optimizer Loop](../05-evaluator-optimizer-loop.md)

Pick a task with an objective bar. Good choices: write a Python function that must pass a given test suite; extract structured data that must validate against a schema; or write copy that must satisfy an explicit rubric. A task where the "critic" can run a real check (tests, a validator) is best.

## Prerequisites

- The agent loop from mod-201.
- For the code/extraction variants: a way to run the check (pytest, a JSON-schema validator).

## Tasks

### 1. Generator and critic

- Define a `generator` agent (produces/revises the artifact) and a `critic` agent with an **adversarial** system prompt ("your job is to find what's wrong"), a different persona from the generator.
- Where possible, give the critic a real tool — run the tests, validate the schema — so its verdict is grounded in evidence, not opinion.

### 2. Structured verdict

- The critic must return a `Verdict`: `{passes: bool, score: int, issues: [...], must_fix: [...]}`. No prose verdicts.
- Define the pass condition explicitly and make it **reachable**: e.g. `score >= 8 and not must_fix`.

### 3. The loop

- Implement the loop: generate → evaluate → if not passing, revise with the critic's `issues` → repeat.
- Add a **hard round cap** (`max_rounds = 4`).
- Add a **no-progress exit**: if the score doesn't improve for 2 rounds, stop and return the best draft so far, flagged as not-passed.

### 4. Show both endings

- Run a task it can **pass** within the cap. Show the score climbing and the loop exiting on the pass condition.
- Run a task it **can't** pass (e.g., an impossible test, or a contradictory rubric). Show it exiting via the round cap or no-progress rule — not spinning forever — and returning the best effort flagged.

### 5. Compare to single-pass and to a deterministic check

- Run the same task with a single generator call (no critic). Compare quality and cost.
- If your task has a deterministic check (tests/validator), use *that* as the evaluator instead of a critic agent for one run, and compare cost.

## Starter guidance

```python
from pydantic import BaseModel

class Verdict(BaseModel):
    passes: bool
    score: int            # 1–10 against an explicit rubric
    issues: list[str]
    must_fix: list[str]

async def evaluate_optimize(task: str, max_rounds=4) -> str:
    draft = await run_agent(system=GENERATOR, task=task)
    best, best_score, no_progress = draft, -1, 0
    for r in range(max_rounds):
        v: Verdict = await run_agent(system=CRITIC, task=task,
                                     artifact=draft, response_model=Verdict)
        if v.passes:
            return draft
        if v.score > best_score:
            best, best_score, no_progress = draft, v.score, 0
        else:
            no_progress += 1
            if no_progress >= 2:
                return best   # converged / stuck
        draft = await run_agent(system=GENERATOR,
                                task=task + "\nFix:\n" + "\n".join(v.issues),
                                prior=draft)
    return best
```

## Acceptance criteria

You can demonstrate that:

- The critic returns a **structured** `Verdict` with an explicit, reachable pass condition.
- The passable task exits on the pass condition with the score having improved across rounds.
- The unpassable task **terminates** via the round cap or no-progress exit and returns a flagged best-effort — it never loops forever.
- Your comparison shows what the critic loop bought (or didn't) over a single pass, and what a deterministic check saved when applicable.

## Reflection

In `NOTES.md`:

1. What pass condition did you start with, and did it ever fail to converge? How did you make it reachable?
2. When did the critic and generator disagree in a way that stalled progress? How did your no-progress exit handle it?
3. For your task, was an LLM critic worth it, or would a test/validator alone have been enough?

## Stretch goals

- Make the critic emit a per-criterion rubric breakdown and show the generator fixing the lowest-scoring criterion first.
- Add a `human` approval gate as the final evaluator for a task where taste matters.
- Track score-vs-round across many runs and plot the convergence curve; tune `max_rounds` from the data.
