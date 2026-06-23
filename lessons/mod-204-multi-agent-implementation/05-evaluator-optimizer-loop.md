# Chapter 5 — The Evaluator-Optimizer Loop

The patterns so far split work across agents *in space*. The evaluator-optimizer loop splits it *in time*: a **generator** produces a candidate, a **critic** judges it against a bar, and the generator revises — iterating until the work is good enough or the loop gives up.

```
        ┌──────────────┐    draft     ┌────────────┐
  task ─▶│  generator   │ ───────────▶ │  evaluator │
        │  (optimizer) │ ◀─────────── │  (critic)  │
        └──────────────┘  feedback     └────────────┘
              ▲                              │
              └──────── revise ──────────────┘
                 until pass OR budget hit
```

## Why two agents instead of one

A single agent asked to "write it and check it" tends to rubber-stamp its own work — it's primed by what it just produced. Separating the roles helps because:

- The **critic carries a different system prompt**: not "help the user," but "find what's wrong." Adversarial framing surfaces problems a self-review glosses over.
- The critic sees the output **fresh**, without the generator's reasoning that rationalizes it.
- You can give the critic **different tools** — run the tests, fetch the spec, check the data — so its judgment is grounded in evidence, not vibes.

This mirrors how the [Stop-hook verification pattern](../mod-205-evaluation-observability/README.md) works for a single agent, lifted to a two-agent loop.

## The loop

```python
async def evaluate_optimize(task: str, max_rounds: int = 4) -> str:
    draft = await run_agent(system=GENERATOR, task=task)
    for round_i in range(max_rounds):
        verdict = await run_agent(
            system=CRITIC, task=task, artifact=draft,
            response_model=Verdict,           # {passes: bool, score: int, issues: [...]}
        )
        if verdict.passes:
            return draft
        draft = await run_agent(
            system=GENERATOR,
            task=f"{task}\n\nRevise to fix:\n" + "\n".join(verdict.issues),
            prior=draft,
        )
    return draft   # budget exhausted — return best effort, flagged as not-passed
```

Force the critic to return a **structured verdict**, not prose:

```python
class Verdict(BaseModel):
    passes: bool
    score: int                 # 1–10 against an explicit rubric
    issues: list[str]          # specific, actionable — "X is wrong because Y"
    must_fix: list[str]        # blocking subset of issues
```

## Not looping forever

The single biggest failure of this pattern is non-termination — burning budget while score oscillates. Defend on three axes:

1. **A hard round cap** (`max_rounds`). Always. This is your backstop.
2. **A pass condition the critic can actually reach.** A vague rubric ("is it good?") never converges. Give the critic concrete, checkable criteria. "Score ≥ 8 *and* `must_fix` is empty" beats "score = 10."
3. **A no-progress exit.** If the score doesn't improve for two rounds, stop — more iterations won't help, and the issues list is probably contradictory or unfixable by this generator. Return the best draft so far, flagged.

```python
if round_i >= 1 and verdict.score <= best_score:
    no_progress += 1
    if no_progress >= 2:
        return best_draft   # converged or stuck — stop paying
```

## Where it shines, where it doesn't

**Use it** when quality is judgeable against a bar and revision is cheaper than getting it right first try: code that must pass tests, copy that must hit a brief, a plan that must satisfy constraints, structured extraction that must validate.

**Skip it** when there's no objective bar (the critic just relitigates taste), when one pass is already reliably good (you're paying 4× for nothing), or when a deterministic check would do — if a linter or test suite can be the "critic," use *that* as the evaluator and save a model call.

## Key takeaways

- Generator + adversarial critic beats single-agent self-review because the roles, framing, and tools differ.
- Force a **structured verdict** with an explicit, reachable pass condition.
- **Always** bound it: hard round cap, a real pass criterion, and a no-progress exit — and prefer a deterministic check (tests/linter) as the evaluator when one exists.
