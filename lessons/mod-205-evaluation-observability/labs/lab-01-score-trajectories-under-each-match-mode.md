# Lab 1 — Score Mocked Trajectories Under Each Match Mode

**Duration:** ~40 minutes
**Covers:** [Chapter 1 — Trajectory and Tool-Call Evaluation](../01-trajectory-tool-call-eval.md)

## Objective

Chapter 1 says the loosest match mode that still catches real bugs is the right one — and shows why the stricter modes produce false failures on agents that take valid longer paths. Before Exercise 01 asks you to capture trajectories from a real agent loop, spend 40 minutes driving **hand-authored** trajectories through each of Chapter 1's match modes so you can *see* each mode's strictness in isolation.

By the end of the lab you will:

- Port the `Step`, `Trajectory`, and `in_order_subset` code from Chapter 1 verbatim.
- Run five scripted trajectories against the same `expected` tool sequence under three match modes (exact, in-order subset, set) and read off which bugs each mode catches vs. which it falsely flags.
- Add one **reference-free invariant** from the chapter — the "same tool called 3 times in a row" repeat detector — and watch it flag a looping trajectory the match modes alone miss.

No API calls, no tool implementations, no agent loop. The trajectories are the input. The point is to feel the Chapter 1 match-mode spectrum without any of Exercise 01's instrumentation work.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 1](../01-trajectory-tool-call-eval.md) read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-01 && cd lab-01
```

No external packages required — pure Python stdlib.

## Step 1 — Port the Chapter 1 shapes

Translate Chapter 1's dataclasses and helper directly. No invention — the lab works against the same shapes Exercise 01 will.

Save as `trajectory.py`:

```python
from dataclasses import dataclass, field


@dataclass
class Step:
    kind: str                       # "reason" | "tool_call" | "tool_result" | "answer"
    tool: str | None = None
    args: dict = field(default_factory=dict)
    output: str | None = None


@dataclass
class Trajectory:
    task: str
    steps: list[Step] = field(default_factory=list)
    final: str | None = None


def tool_calls(t: Trajectory) -> list[str]:
    """Extract the ordered list of tool names from a trajectory."""
    return [s.tool for s in t.steps if s.kind == "tool_call"]


def in_order_subset(actual: list[str], expected: list[str]) -> bool:
    """Chapter 1's pragmatic default: every expected tool appears in the
    right relative order; benign extra steps are allowed."""
    it = iter(actual)
    return all(tool in it for tool in expected)
```

Two details worth noticing:

- `Step.kind` is a discriminator string, not a subclass hierarchy. Chapter 1 keeps it flat because the trajectory is a *record*, not a state machine.
- `in_order_subset` is four lines because it uses the `iter`/`in` trick: each `tool in it` consumes items until a match, so the next lookup starts *after* the previous match — which is exactly what "in order" means.

## Step 2 — The three match modes side by side

Chapter 1 lists exact, in-order subset, set, and single-step precision/recall. Implement the first three — they're the ones you choose between when pinning expectations on a case.

Save as `match_modes.py`:

```python
from trajectory import tool_calls, in_order_subset, Trajectory


def exact_match(actual: list[str], expected: list[str]) -> bool:
    """Step sequence equals the reference exactly. Brittle; Chapter 1 reserves
    this for tightly scripted flows."""
    return actual == expected


def set_match(actual: list[str], expected: list[str]) -> bool:
    """The *set* of tools used equals the expected set; order ignored.
    Use when order genuinely doesn't matter."""
    return set(actual) == set(expected)


def score(t: Trajectory, expected: list[str]) -> dict[str, bool]:
    calls = tool_calls(t)
    return {
        "exact":           exact_match(calls, expected),
        "in_order_subset": in_order_subset(calls, expected),
        "set":             set_match(calls, expected),
    }
```

Chapter 1 tells you to "pick the loosest mode that still catches real bugs." The next step gives you the trajectories that let you see what "loose enough" and "too loose" look like concretely.

## Step 3 — Five hand-authored trajectories, one expected sequence

Each trajectory is built to land in a specific cell of the strictness grid — a *valid variation*, a *benign extra step*, a *reorder bug*, a *missing-tool bug*, or a *loop bug*. All are scored against the same expected `["search", "calc"]`.

Save as `cases.py`:

```python
from trajectory import Step, Trajectory


def _tc(tool: str, **args) -> Step:
    """Shorthand for a tool-call step."""
    return Step(kind="tool_call", tool=tool, args=args)


EXPECTED = ["search", "calc"]


CASES: dict[str, Trajectory] = {
    # 1. Canonical path — matches every mode.
    "canonical": Trajectory(
        task="2 + 2 after a lookup",
        steps=[
            Step(kind="reason"),
            _tc("search", q="2+2"),
            Step(kind="tool_result", output="use calculator"),
            _tc("calc", expr="2+2"),
            Step(kind="tool_result", output="4"),
            Step(kind="answer"),
        ],
        final="4",
    ),

    # 2. Benign extra step — a `lookup` between search and calc.
    #    Chapter 1: in-order subset should pass this; exact should not.
    "extra_benign_step": Trajectory(
        task="2 + 2 after a lookup",
        steps=[
            _tc("search", q="2+2"),
            _tc("lookup", key="arithmetic"),
            _tc("calc", expr="2+2"),
        ],
        final="4",
    ),

    # 3. Reorder bug — calc was called before search. A real latent bug
    #    even if the answer happens to be right.
    "reorder_bug": Trajectory(
        task="2 + 2 after a lookup",
        steps=[
            _tc("calc", expr="2+2"),
            _tc("search", q="2+2"),
        ],
        final="4",
    ),

    # 4. Missing-tool bug — `calc` was never called at all. A single-tool
    #    trajectory is strictly a dropped step.
    "missing_tool": Trajectory(
        task="2 + 2 after a lookup",
        steps=[_tc("search", q="2+2")],
        final="4",
    ),

    # 5. Loop bug — correct tools in order, but `calc` repeats 3 times.
    #    Every match mode passes this; only a reference-free invariant catches it.
    "loop_bug": Trajectory(
        task="2 + 2 after a lookup",
        steps=[
            _tc("search", q="2+2"),
            _tc("calc", expr="2+2"),
            _tc("calc", expr="2+2"),
            _tc("calc", expr="2+2"),
        ],
        final="4",
    ),
}
```

Notice: every `final` is `"4"` — the answer is "correct" in all five cases. Chapter 1's whole premise is that **outcome eval alone would pass all of these**. That's the setup.

## Step 4 — Run the grid

Save as `run_match_modes.py`:

```python
from cases import CASES, EXPECTED
from match_modes import score


print(f"expected tools: {EXPECTED}")
print(f"{'case':<22} {'exact':>7} {'in_ord':>7} {'set':>7}")
print("-" * 46)
for name, traj in CASES.items():
    s = score(traj, EXPECTED)
    def mark(b: bool) -> str: return "pass" if b else "FAIL"
    print(f"{name:<22} {mark(s['exact']):>7} "
          f"{mark(s['in_order_subset']):>7} {mark(s['set']):>7}")
```

Run:

```bash
python run_match_modes.py
```

**Expected output:**

```
expected tools: ['search', 'calc']
case                     exact  in_ord     set
----------------------------------------------
canonical                 pass    pass    pass
extra_benign_step         FAIL    pass    pass
reorder_bug               FAIL    FAIL    pass
missing_tool              FAIL    FAIL    FAIL
loop_bug                  FAIL    pass    pass
```

Read the table one row at a time:

- **canonical** — passes everywhere; this is the shape the expectation was written for.
- **extra_benign_step** — exact fails on a *valid* variation (the agent took a longer path to the same answer). This is the false-failure mode Chapter 1 warns about: a stricter mode would block every PR that introduced a new helpful intermediate step.
- **reorder_bug** — exact and in-order-subset both catch it; set mode misses it because the *tools* used are the same two. If the required sequence is `authenticate → fetch`, set mode is the wrong choice.
- **missing_tool** — every mode catches it. Chapter 1: tool-call correctness means every expected call must appear.
- **loop_bug** — every mode passes it. The expected tools were called in the right order; the problem is that `calc` was called three times. **This is the case the match modes *cannot* catch — only a reference-free invariant can.**

## Step 5 — Add a reference-free invariant for the loop bug

Chapter 1: "assert it never repeated the identical call three times in a row." Code it, run it over the same five cases, and watch it name the one case the match modes missed.

Save as `invariants.py`:

```python
from trajectory import Trajectory


def repeated_identical_call(t: Trajectory, n: int = 3) -> tuple[str, dict] | None:
    """Return (tool, args) if the same (tool, args) tool_call appears `n` times
    in a row. None otherwise. Chapter 1's cheap, reference-free loop detector."""
    run_key, run_len = None, 0
    for step in t.steps:
        if step.kind != "tool_call":
            run_key, run_len = None, 0
            continue
        key = (step.tool, tuple(sorted(step.args.items())))
        if key == run_key:
            run_len += 1
            if run_len >= n:
                return step.tool, step.args
        else:
            run_key, run_len = key, 1
    return None
```

Notice three Chapter 1 moves baked in:

- **Compare `(tool, args)`**, not just `tool`. A worker that calls `search` for three different queries is not looping; a worker that calls `search` for the identical query three times is.
- **Reset the run on any non-tool-call step.** A `reason` or `tool_result` between calls means the agent *did* react — not a tight loop.
- **No reference needed.** This check scales to every task in the suite without hand-written expectations. Chapter 1: "reference-free invariants catch loops, argument bugs, and hallucinated citations cheaply, and they scale to any task."

Save as `run_invariants.py`:

```python
from cases import CASES
from invariants import repeated_identical_call


print(f"{'case':<22} {'repeat?':>18}")
print("-" * 42)
for name, traj in CASES.items():
    hit = repeated_identical_call(traj, n=3)
    label = f"LOOP: {hit[0]}({hit[1]})" if hit else "ok"
    print(f"{name:<22} {label:>18}")
```

Run:

```bash
python run_invariants.py
```

**Expected output:**

```
case                              repeat?
------------------------------------------
canonical                              ok
extra_benign_step                      ok
reorder_bug                            ok
missing_tool                           ok
loop_bug               LOOP: calc({'expr': '2+2'})
```

Only `loop_bug` is named — exactly the case every match mode passed in Step 4. This is why Chapter 1 pairs match-mode scoring with reference-free invariants: *they catch disjoint failure modes*. A trajectory eval that uses only one of them has a blind spot.

## Step 6 — Break the invariant's assumption and feel it

The invariant resets its run counter on any non-tool-call step. That is a deliberate Chapter 1 choice ("no identical call three times *in a row*"), but it has an edge: a looping agent that interleaves a cosmetic `reason` step between each repeated call will slip past.

In `cases.py`, add one more case:

```python
    # 6. Masked loop — same identical calc(2+2) three times, but with a
    #    dummy `reason` step between each. The invariant from Chapter 1 as
    #    written will miss this.
    "masked_loop": Trajectory(
        task="2 + 2 after a lookup",
        steps=[
            _tc("search", q="2+2"),
            _tc("calc", expr="2+2"),
            Step(kind="reason"),
            _tc("calc", expr="2+2"),
            Step(kind="reason"),
            _tc("calc", expr="2+2"),
        ],
        final="4",
    ),
```

Rerun `python run_invariants.py`. The `masked_loop` case shows `ok` — the invariant missed it.

**Why this matters:** every reference-free invariant encodes an implicit theory of what a bug looks like. The "in a row" theory catches tight loops but not loops that interleave reasoning. In your real suite you'd pair it with a second invariant — e.g., "the same `(tool, args)` tuple appeared more than K times in the whole trajectory, regardless of adjacency." Chapter 1's point is not that one invariant is enough; it's that **cheap invariants compose** and you should build a small battery of them.

Revert the extra case (or keep it as a note on the invariant's blind spot).

**Checkpoint:** without peeking, you can state:

1. Which of the three match modes fails on the `extra_benign_step` case, and why Chapter 1 calls that a *false failure* rather than a real bug.
2. Which match mode would pass `reorder_bug` incorrectly, and the one-sentence rule for when that mode is safe to use anyway.
3. Why the `loop_bug` case needs a reference-free invariant and could never be caught by any mode in Step 4 — and the one additional invariant you'd add to cover `masked_loop`.

## Wrap-up

Carry this harness into [Exercise 01](../exercises/exercise-01-trajectory-eval-implementation.md): the `Step`, `Trajectory`, `tool_calls`, `in_order_subset`, match-mode functions, and `repeated_identical_call` invariant are all exactly the shapes the exercise asks you to build. The work left for the exercise is the thing this lab skipped on purpose: **instrumenting the loop to produce the `Trajectory` instead of hand-writing it**, and running the result over a dataset of 5–8 tasks with real agent output. The scoring layer — the part that decides pass/fail — you already have.
