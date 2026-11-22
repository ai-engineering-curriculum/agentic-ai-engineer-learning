# Lab 1 — Feel Specific-Beats-General with a Mock Agent

**Duration:** ~45 minutes
**Covers:** [Chapter 1 — What a Harness Is](../01-what-is-a-harness.md), [Chapter 2 — Guides: Context, Instructions, and Skills](../02-guides-context-and-skills.md)

## Objective

Chapters 1 and 2 argue that a short, specific, checkable guide beats a long, general one — and that a sensor that keeps firing should become a guide. Before [Exercise 01](../exercises/exercise-01-harness-a-repository.md) asks you to write an `AGENTS.md`, one skill, and a permission policy against a real repo and real agent, spend 45 minutes driving a **scripted mock agent** through the same task under three different instruction files so the mechanic is visible without spending a token.

By the end of the lab you will:

- Drive a tiny mock agent against the same task with (a) no guide, (b) a weak, general guide, (c) a strong, specific guide that names files and commands, and watch pass rate climb.
- Feel the Chapter 2 maxim "every line must earn its place" — cut the weak guide down to 5 lines and see pass rate hold.
- Watch a *sensor* catch a recurring mistake across trials, then convert it into a *guide* and watch the mistake stop happening in the first place.

The mock agent is deterministic (seeded) and plays no model tokens. The *shapes* of guide vs. sensor are what you're here to feel.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 1](../01-what-is-a-harness.md) and [Chapter 2](../02-guides-context-and-skills.md) read.
- No external packages — pure stdlib.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-01 && cd lab-01
```

## Step 1 — A scripted mock agent

A real agent reads the instruction file, picks a tool call per turn, and sometimes gets things wrong. The mock agent does the same shape with **deterministic** choices keyed by the words it sees in the guide. That lets you compare three guides side by side with the same seed — no run-to-run noise.

Save as `mock_agent.py`:

```python
import random
from dataclasses import dataclass


CANDIDATE_ACTIONS = [
    "edit:src/tool/cli.py",            # correct for "add a subcommand"
    "edit:src/tool/commands/new.py",   # correct for "add a subcommand"
    "edit:README.md",                  # wrong file
    "edit:docs/cli.md",                # wrong file
    "run:make check",                  # correct done-check
    "run:pytest -k anything",          # partial — misses lint/typecheck
    "say:done",                        # premature claim
]


@dataclass(frozen=True)
class TrialResult:
    actions: list[str]
    passed: bool
    reason: str


def _score(guide: str, action: str) -> int:
    """Higher = more likely to pick this action. The guide's words tilt the model."""
    s = 1
    low = guide.lower()
    if "cli.py" in action and ("cli.py" in low or "subcommand" in low):
        s += 4
    if "commands/" in action and "commands/" in low:
        s += 3
    if "make check" in action and "make check" in low:
        s += 5
    if "say:done" in action and "done" in low and "make check" not in low:
        s += 2  # vague guides reward premature claims
    if "readme" in action.lower() and "readme" in low:
        s += 1
    return s


def run_agent(task: str, guide: str, seed: int, max_turns: int = 5) -> TrialResult:
    """Tiny reason-act loop. Picks actions weighted by the guide's words."""
    rng = random.Random(seed)
    actions: list[str] = []
    for _ in range(max_turns):
        weights = [_score(guide, a) for a in CANDIDATE_ACTIONS]
        action = rng.choices(CANDIDATE_ACTIONS, weights=weights, k=1)[0]
        actions.append(action)
        if action == "say:done":
            break

    touched_right_file = any("cli.py" in a or "commands/" in a for a in actions)
    ran_full_check = any("make check" in a for a in actions)
    if touched_right_file and ran_full_check:
        return TrialResult(actions, True, "edited the CLI and ran make check")
    if not touched_right_file:
        return TrialResult(actions, False, "never edited the CLI files")
    return TrialResult(actions, False, "edited CLI but never ran make check")
```

Two details worth noticing:

- The scoring function is the stand-in for everything the real model does. Specific words in the guide (file paths, commands) **tilt** the agent toward the right action; vague words (`done`, `tests`) don't. This is a cartoon of Chapter 2's "specific beats general".
- `max_turns` and the `say:done` short-circuit mimic a real harness's turn budget: the agent can quit early, correctly or not.

## Step 2 — Three guides, same task

Save as `guides.py`:

```python
NO_GUIDE = ""

WEAK_GUIDE = """\
# tool

Write clean, well-tested code. Follow our conventions. Be careful
with the CLI. Run the tests when you're done.
"""

STRONG_GUIDE = """\
# tool

A small CLI for exporting reports.

## Commands
- Check (run before you say you're done): `make check`
- Single test: `pytest tests/commands/test_<name>.py`

## Map
- `src/tool/cli.py` — argparse; subcommands register here
- `src/tool/commands/` — one file per subcommand

## Adding a subcommand
1. Create `src/tool/commands/<name>.py` with `register(subparsers)`.
2. Add the import + register call in `src/tool/cli.py`, keeping the list alphabetical.
3. Run `make check`.
"""
```

Three details worth noticing:

- `WEAK_GUIDE` is a textbook "general advice" guide: it uses the right nouns (*CLI*, *tests*, *done*) but doesn't name files, commands, or the check. Chapter 2 calls this out in the weak-vs-strong table.
- `STRONG_GUIDE` is under 15 lines and does all three Chapter 2 things: exact commands, directory map, pointer to the procedure.
- Nothing in either guide copies the architecture doc or restates the linter. Neither would earn its place; the agent would get both wrong without the file.

## Step 3 — A scorer across trials

Real harness evals run several trials per task so you're measuring an effect, not a dice roll (Chapter 4 previews this). Even for this lab, one trial each would tell you almost nothing. Run 30.

Save as `score.py`:

```python
from mock_agent import run_agent
from guides import NO_GUIDE, WEAK_GUIDE, STRONG_GUIDE


def pass_rate(guide: str, n: int = 30) -> tuple[float, list]:
    results = [run_agent("add a new `export-json` subcommand", guide, seed=i) for i in range(n)]
    rate = sum(1 for r in results if r.passed) / n
    return rate, results


if __name__ == "__main__":
    for name, guide in [("none", NO_GUIDE), ("weak", WEAK_GUIDE), ("strong", STRONG_GUIDE)]:
        rate, results = pass_rate(guide)
        failed_reasons = [r.reason for r in results if not r.passed]
        print(f"{name:7s} pass_rate={rate:.2f}  n_fail={len(failed_reasons)}")
        if failed_reasons:
            top = max(set(failed_reasons), key=failed_reasons.count)
            print(f"        most common failure: {top}")
```

Run it:

```bash
python score.py
```

**Expected output** (exact numbers vary; shape is the point):

```
none    pass_rate=0.07  n_fail=28
        most common failure: never edited the CLI files
weak    pass_rate=0.13  n_fail=26
        most common failure: edited CLI but never ran make check
strong  pass_rate=0.63  n_fail=11
        most common failure: edited CLI but never ran make check
```

Three things to notice:

- **The weak guide barely beats nothing.** It uses the right nouns but doesn't name files or commands; the agent still guesses. Chapter 2's "weak vs. strong" table in words.
- **The strong guide moves the needle a lot.** The *same* model (the same scoring function!) does much better because the guide named files and the check command — precisely what the agent needs to pick an action.
- **The dominant failure mode changed.** With no guide the agent doesn't edit the right file; with the strong guide the remaining failures are the agent forgetting to run `make check`. That's your next bug.

## Step 4 — Turn the recurring sensor into a guide

Chapter 1: *"A sensor that keeps firing should become a guide."* Right now `make check` is a sensor the agent sometimes remembers to run. Make it harder to forget by strengthening the guide line about it.

Edit `guides.py` and tighten the strong guide's check line:

```python
STRONG_GUIDE = """\
# tool

A small CLI for exporting reports.

## Commands
- Check (run before you say you're done; `make check` must exit 0): `make check`
- Single test: `pytest tests/commands/test_<name>.py`

## Map
- `src/tool/cli.py` — argparse; subcommands register here
- `src/tool/commands/` — one file per subcommand

## Done condition
You are not done until `make check` passes. Do not say "done" otherwise.

## Adding a subcommand
1. Create `src/tool/commands/<name>.py` with `register(subparsers)`.
2. Add the import + register call in `src/tool/cli.py`, keeping the list alphabetical.
3. Run `make check`.
"""
```

Re-run:

```bash
python score.py
```

**Expected shape:**

```
strong  pass_rate=0.80  n_fail=6
        most common failure: edited CLI but never ran make check
```

The "done condition" line (named, specific, checkable) tilts the scorer away from the premature `say:done` action. In the real world that same line is why your stop-gate hook from [Exercise 02](../exercises/exercise-02-guardrail-hooks.md) will fire less often: a guide that works upstream of the sensor saves the sensor some work.

## Step 5 — Every line must earn its place

Chapter 2: *"For every line, ask 'would the agent get this wrong without it?' If not, delete it."* Try it. Delete the directory map from `STRONG_GUIDE` and re-run `score.py`.

**Expected shape:**

```
strong  pass_rate=0.35  n_fail=20
        most common failure: never edited the CLI files
```

Pass rate falls sharply. That line was earning its place — without the map, the agent doesn't know which files are "the CLI files" and starts editing the README.

Now delete, instead, the "Write clean, well-tested code"-style sentence you added just in case — oh wait, the strong guide doesn't have one. That's the point. Go read your real-project `AGENTS.md` and find one line that, if removed, the agent would get the same things right. That line is costing you tokens every session.

## Step 6 — Checkpoint

Without peeking, you can state:

1. Why the weak guide barely beats having no guide, in Chapter 2's own words — and the two changes that turn it into a strong guide.
2. What "sensor becomes a guide" looked like in Step 4: the sensor was `make check` not getting run; the guide was the "Done condition" line that tilted the agent away from premature claims.
3. One line in your real `AGENTS.md` (or a template you've seen) that *doesn't* earn its place by the Step 5 test.

## Wrap-up

Carry the mindset — but not the toy scoring function — into [Exercise 01](../exercises/exercise-01-harness-a-repository.md), which asks you to:

- Write a real `AGENTS.md` under 80 lines against a real repo.
- Add one skill with a `description` that works as a trigger (Chapter 2).
- Set a permission policy (Chapter 3 preview) and a single `make check` command.
- Run three real tasks in a real agent before and after, and compare.

The gap this lab closes: in Exercise 01 you'll be tempted to write 300 lines of `AGENTS.md`. You now know, from Step 5, that most of them probably wouldn't change a pass rate.

### Stretch

- Add a fourth guide: `STRONG_GUIDE` plus a skill-trigger line (`"Adding a subcommand: use the add-subcommand skill"`) and a mock `SKILL.md` body that the mock agent loads only when the task mentions "subcommand". You should see pass rate rise again *without* lengthening what loads every session — Chapter 2's progressive disclosure made concrete.
- Swap the scoring function for one that only rewards file paths named in the guide. This is closer to how real models behave: naming a file in the instruction is usually the single highest-value edit you can make.
- Run `score.py` with `n=300` and `n=5` on the same two guides and compare the spread across repeats. (Preview of Chapter 4: three trials isn't many.)
