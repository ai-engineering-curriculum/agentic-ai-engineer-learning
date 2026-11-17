# Lab 3 — Drive the Evaluator-Optimizer Loop with a Scripted Critic

**Duration:** ~45 minutes
**Covers:** [Chapter 5 — The Evaluator-Optimizer Loop](../05-evaluator-optimizer-loop.md)

## Objective

Before Exercise 04 asks you to wire a real generator + critic against a model and a task with an objective bar, build the loop's **control flow** against a scripted mock critic. The hard part of this pattern is not the model — it's making sure the loop *stops* for the right reason. In 45 minutes you will exercise all three of Chapter 5's termination paths on demand:

- Exit on the pass condition.
- Exit on `max_rounds`.
- Exit on the no-progress rule.

No API calls. A scripted `MockCritic` makes each scenario deterministic, so you can run the loop a hundred times and get the same answer.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 5](../05-evaluator-optimizer-loop.md) read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
pip install pydantic
mkdir lab-03 && cd lab-03
```

## Step 1 — The `Verdict` schema

Chapter 5 insists on a **structured** verdict. Translate it to pydantic verbatim.

Save as `verdict.py`:

```python
from pydantic import BaseModel, Field


class Verdict(BaseModel):
    passes: bool
    score: int = Field(ge=1, le=10)
    issues: list[str]
    must_fix: list[str]
```

The `score` is bounded to the explicit 1–10 rubric from the chapter. A critic that tries to return `0` or `11` is rejected at the boundary — exactly as a real critic should be.

## Step 2 — A scripted mock critic

A scripted critic lets you *choose* which termination path a given run will take. This is the thing a real LLM critic can't give you: determinism.

Save as `mock_critic.py`:

```python
from verdict import Verdict


class MockCritic:
    """Cycles through pre-canned verdicts. When the script is exhausted, the
    last verdict repeats — which is what lets you simulate a stuck critic."""

    def __init__(self, script: list[Verdict]):
        if not script:
            raise ValueError("script cannot be empty")
        self._script = list(script)
        self._i = 0

    def judge(self, draft: str) -> Verdict:
        v = self._script[min(self._i, len(self._script) - 1)]
        self._i += 1
        return v
```

## Step 3 — A scripted mock generator

Save as `mock_generator.py`:

```python
class MockGenerator:
    """Returns drafts from a script. Each call to revise() advances to the
    next draft, so you can pre-plan the trajectory the loop should see."""

    def __init__(self, drafts: list[str]):
        if not drafts:
            raise ValueError("drafts cannot be empty")
        self._drafts = list(drafts)
        self._i = 0

    def write(self, task: str) -> str:
        self._i = 0
        return self._drafts[0]

    def revise(self, task: str, prior: str, issues: list[str]) -> str:
        self._i = min(self._i + 1, len(self._drafts) - 1)
        return self._drafts[self._i]
```

## Step 4 — The loop, with all three exits

Transcribe Chapter 5's loop into Python, but with every exit named explicitly so you can tell from the output which one fired.

Save as `loop.py`:

```python
from verdict import Verdict
from mock_critic import MockCritic
from mock_generator import MockGenerator


def evaluate_optimize(task: str,
                      gen: MockGenerator,
                      crit: MockCritic,
                      max_rounds: int = 4) -> tuple[str, str, int]:
    """Return (final_draft, exit_reason, rounds_used)."""
    draft = gen.write(task)
    best, best_score, no_progress = draft, -1, 0

    for r in range(max_rounds):
        v: Verdict = crit.judge(draft)
        print(f"  round {r + 1}: score={v.score}, passes={v.passes}, "
              f"issues={len(v.issues)}, must_fix={len(v.must_fix)}")

        if v.passes:
            return draft, "pass-condition", r + 1

        if v.score > best_score:
            best, best_score, no_progress = draft, v.score, 0
        else:
            no_progress += 1
            if no_progress >= 2:
                return best, "no-progress", r + 1

        draft = gen.revise(task=task, prior=draft, issues=v.issues)

    return best, "max-rounds", max_rounds
```

Three details Chapter 5 is strict about:

- `passes` is the first thing checked each round — the only reason we exit "clean."
- The best draft is tracked separately from the current draft, so a worsening revision doesn't replace a good earlier one.
- If both exits fire in the same round (no-progress *and* this was the last allowed round), no-progress wins because it's tested first. Either is a correct terminate-and-flag outcome.

## Step 5 — Scenario A: exit on the pass condition

Save as `main.py`:

```python
from verdict import Verdict
from mock_critic import MockCritic
from mock_generator import MockGenerator
from loop import evaluate_optimize


def scenario(name: str, drafts: list[str], script: list[Verdict]) -> None:
    print(f"\n=== {name} ===")
    gen = MockGenerator(drafts)
    crit = MockCritic(script)
    final, reason, rounds = evaluate_optimize("task", gen, crit)
    print(f"  exit: {reason} after {rounds} rounds")
    print(f"  final: {final!r}")


# A: climbing score, passes on round 3.
scenario(
    "A: converges and passes",
    drafts=["draft v1", "draft v2", "draft v3"],
    script=[
        Verdict(passes=False, score=4, issues=["weak thesis"], must_fix=["weak thesis"]),
        Verdict(passes=False, score=6, issues=["citation gap"], must_fix=[]),
        Verdict(passes=True,  score=9, issues=[],              must_fix=[]),
    ],
)
```

Run:

```bash
python main.py
```

**Expected output:**

```
=== A: converges and passes ===
  round 1: score=4, passes=False, issues=1, must_fix=1
  round 2: score=6, passes=False, issues=1, must_fix=0
  round 3: score=9, passes=True, issues=0, must_fix=0
  exit: pass-condition after 3 rounds
  final: 'draft v3'
```

**Why this matters:** the pass condition fired *before* the round cap. That is the only ending the loop should exit on "clean" — the critic's structured verdict said it's good, and the critic is authoritative.

## Step 6 — Scenario B: no-progress exit

Append to `main.py`:

```python
# B: score oscillates but never climbs — no-progress fires.
scenario(
    "B: stalls, no-progress exit",
    drafts=["draft v1", "draft v2", "draft v3", "draft v4", "draft v5"],
    script=[
        Verdict(passes=False, score=5, issues=["issue X"], must_fix=["X"]),
        Verdict(passes=False, score=5, issues=["issue X"], must_fix=["X"]),  # no-progress #1
        Verdict(passes=False, score=5, issues=["issue X"], must_fix=["X"]),  # no-progress #2 → exit
    ],
)
```

Rerun. **Expected output (appended):**

```
=== B: stalls, no-progress exit ===
  round 1: score=5, passes=False, issues=1, must_fix=1
  round 2: score=5, passes=False, issues=1, must_fix=1
  round 3: score=5, passes=False, issues=1, must_fix=1
  exit: no-progress after 3 rounds
  final: 'draft v1'
```

**Why `draft v1`:** the loop returned the *best* draft, which was the first one — nothing after it scored higher. Chapter 5: "stop — more iterations won't help, and the issues list is probably contradictory or unfixable by this generator."

## Step 7 — Scenario C: max-rounds exit

Append to `main.py`:

```python
# C: climbing score but never reaches pass — max_rounds backstop fires.
scenario(
    "C: slow climber, max-rounds backstop",
    drafts=["draft v1", "draft v2", "draft v3", "draft v4", "draft v5"],
    script=[
        Verdict(passes=False, score=3, issues=["many"], must_fix=["a"]),
        Verdict(passes=False, score=4, issues=["some"], must_fix=["a"]),
        Verdict(passes=False, score=5, issues=["some"], must_fix=["a"]),
        Verdict(passes=False, score=6, issues=["some"], must_fix=["a"]),
    ],
)
```

Rerun. **Expected output (appended):**

```
=== C: slow climber, max-rounds backstop ===
  round 1: score=3, passes=False, issues=1, must_fix=1
  round 2: score=4, passes=False, issues=1, must_fix=1
  round 3: score=5, passes=False, issues=1, must_fix=1
  round 4: score=6, passes=False, issues=1, must_fix=1
  exit: max-rounds after 4 rounds
  final: 'draft v4'
```

**Why this matters:** every round improved. The loop would happily have run for 20 more. The `max_rounds` cap is the backstop that keeps the budget bounded even when the no-progress heuristic is *not* tripping — exactly the "hard round cap. Always. This is your backstop" from Chapter 5.

## Step 8 — Break the Verdict on purpose

Add one more scenario with an invalid verdict and watch the schema reject it before the loop can act on it:

```python
from pydantic import ValidationError

try:
    Verdict(passes=False, score=11, issues=[], must_fix=[])  # score out of range
except ValidationError as e:
    print("\nrejected at boundary:", e.errors()[0]["type"], e.errors()[0]["loc"])
```

**Expected output (appended):**

```
rejected at boundary: less_than_equal ('score',)
```

**Why this matters:** Chapter 5 says "no prose verdicts." Enforcement is at the schema, not the prompt. A critic that wants to return `score=11` can't — the loop never sees an unbounded verdict.

**Checkpoint:** you can draw Chapter 5's loop on a napkin and name — without looking — the three distinct conditions under which it exits, which one your scenario A/B/C each hit, and why the "best draft" is tracked separately from the "current draft."

## Wrap-up

Carry `verdict.py` and `loop.py` into Exercise 04. Replace `MockCritic.judge` with a real critic agent call that returns a `Verdict` via a response schema; replace `MockGenerator.write`/`revise` with real generator agent calls. The control flow — the three termination paths, the best-draft tracking, the schema-enforced bounded score — is unchanged, because it's the whole point of the pattern. Exercise 04's stretch goal of plotting score-vs-round across many runs is easier when the control flow is already paranoid about termination.
