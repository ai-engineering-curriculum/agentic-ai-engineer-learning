# Lab 2 — Hand-Roll a Replay Journal to Feel Durable Execution

**Duration:** ~45 minutes
**Covers:** [Chapter 2 — Durable Execution and Resumption](../02-durable-execution.md)

## Objective

Chapter 2's promise is that completed steps are **recorded** and, on recovery, **replayed from the log** instead of re-executed. Before [Exercise 02](../exercises/exercise-02-durable-execution-temporal.md) asks you to split a real agent into Temporal workflow + activities and prove replay against a live worker, spend 45 minutes building the smallest-possible **replay journal** in pure Python so the semantics are visible without a Temporal server, a cluster, or a single provider token.

By the end of the lab you will:

- Implement a file-backed `Journal` that records each activity's result keyed by a step id.
- Implement an `@activity` wrapper that checks the journal before running — and a `run_workflow` function that replays from the log on restart.
- Kill the process mid-run and restart it; watch the "expensive" model-like activity call its underlying function **zero times** on the second run while the workflow fast-forwards to where it stopped.
- See first-hand that an activity that was **interrupted mid-flight** (not completed) retries — and that only an idempotency key keeps its external side effect from firing twice.

This is a *concept* lab. The real Temporal SDK does more (deterministic schedulers, retries with backoff, timeouts, histories you can view), but the replay behavior you'll feel here is the mechanic those features are built on.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 2](../02-durable-execution.md) read.
- No external packages — pure stdlib.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-02 && cd lab-02
```

## Step 1 — A file-backed journal

The journal is a JSON-lines file keyed by `step_id`. On process start we load everything already recorded; each successful activity call appends a new line.

Save as `journal.py`:

```python
import json
import os
from pathlib import Path


class Journal:
    """Append-only step log, keyed by step_id.

    Writing is fsync'd so a kill -9 after a successful activity doesn't lose
    the record. Reading replays the file in order."""

    def __init__(self, path: str | Path):
        self.path = Path(path)
        self._results: dict[str, object] = {}
        if self.path.exists():
            with self.path.open() as f:
                for line in f:
                    entry = json.loads(line)
                    self._results[entry["step_id"]] = entry["result"]

    def has(self, step_id: str) -> bool:
        return step_id in self._results

    def get(self, step_id: str):
        return self._results[step_id]

    def record(self, step_id: str, result) -> None:
        if step_id in self._results:
            raise RuntimeError(f"step {step_id} already recorded")
        self._results[step_id] = result
        with self.path.open("a") as f:
            f.write(json.dumps({"step_id": step_id, "result": result}) + "\n")
            f.flush()
            os.fsync(f.fileno())
```

Three details worth noticing:

- **Append-only, fsync'd.** A real durable engine writes the history durably before returning control to the workflow. The `flush()` + `fsync()` is the stdlib-only version: a crash after an activity's `record` call will still find that line on restart.
- **Replay is read-then-skip.** The journal doesn't *re-execute* anything itself — it just reports "I already know the result of this step id." The workflow is the one that decides to skip.
- **`record` is strict.** If the workflow asks to record the same `step_id` twice, that's a programming bug (non-deterministic workflow producing a different step graph on replay), not a legitimate retry.

## Step 2 — Activity wrapper that checks the journal first

Save as `activity.py`:

```python
from journal import Journal


def activity(step_id: str, journal: Journal, fn, *args, **kwargs):
    """The core move: if the journal has a recorded result for this step_id,
    return it without running fn. Otherwise run fn and record the result."""
    if journal.has(step_id):
        print(f"  [{step_id}] REPLAY (not calling fn)")
        return journal.get(step_id)
    print(f"  [{step_id}] RUN")
    result = fn(*args, **kwargs)
    journal.record(step_id, result)
    return result
```

This is five lines of code and it is the whole Chapter 2 mechanic: a wrapper that reads the log first and only runs the real function when the log is silent. Everything the Temporal SDK layers on top (retries, timeouts, workers, signals) assumes this.

## Step 3 — A "model call" with a visible side effect

A real model call is expensive and side-effecting (billed, metered, sometimes logged in the provider's dashboard). We fake the expense with a counter and a sleep so a replay that *didn't* run the function has a bright, unmistakable fingerprint: the counter didn't go up.

Save as `fake_model.py`:

```python
import time
from pathlib import Path


COUNTER = Path("model_calls.count")


def call_model(prompt: str, delay: float = 1.0) -> str:
    """Pretend to spend tokens. Bumps a persistent counter so you can see
    across restarts whether the function body actually ran."""
    n = int(COUNTER.read_text()) if COUNTER.exists() else 0
    COUNTER.write_text(str(n + 1))
    time.sleep(delay)
    return f"[answer to: {prompt[:30]}]"
```

The counter is persisted to a file so the restart in Step 6 can read it. In a real system this role is played by the provider's billing dashboard.

## Step 4 — The workflow

A workflow is a function that calls activities in a deterministic order. Deterministic here means: given the same inputs and the same recorded history, the same sequence of `(step_id, fn)` calls happens on every run. That's what makes replay safe.

Save as `workflow.py`:

```python
import sys

from activity import activity
from fake_model import call_model
from journal import Journal


def research_agent(journal: Journal, question: str) -> str:
    """A tiny multi-step agent. Three activities: plan, search, synthesize."""
    plan = activity("plan", journal, call_model, f"Plan steps for: {question}")
    search = activity("search", journal, call_model, f"Execute plan: {plan}")
    answer = activity("synthesize", journal, call_model, f"Synthesize: {search}")
    return answer


if __name__ == "__main__":
    question = sys.argv[1] if len(sys.argv) > 1 else "what is the capital of France?"
    journal = Journal("run-42.journal")
    print(f"running workflow against {journal.path}")
    answer = research_agent(journal, question)
    print(f"final answer: {answer}")
```

Notice what the workflow function **doesn't** do: it never reads a clock, never randomizes, never touches the network itself. All of that lives in the activity it dispatches to. Chapter 2: *"No direct I/O, no `random`, no wall-clock `time.now()` inside a workflow."* Any of those would break replay — the workflow would make a different decision on recovery than it made the first time.

## Step 5 — Run it end to end

Start from a clean slate:

```bash
rm -f run-42.journal model_calls.count
python workflow.py
```

**Expected output (takes ~3 s):**

```
running workflow against run-42.journal
  [plan] RUN
  [search] RUN
  [synthesize] RUN
final answer: [answer to: Synthesize: [answer to: Execute]
```

Check the counter and the journal:

```bash
cat model_calls.count
cat run-42.journal
```

**Expected:**

```
3
{"step_id": "plan", "result": "[answer to: Plan steps for: what is the ca"}
{"step_id": "search", "result": "[answer to: Execute plan: [answer to: Plan"}
{"step_id": "synthesize", "result": "[answer to: Synthesize: [answer to: Execu"}
```

Three activities ran, three model "calls" were billed, three lines appeared in the journal. Clean baseline.

## Step 6 — Replay a complete run

Rerun the workflow *without* clearing the journal:

```bash
python workflow.py
```

**Expected output (instant):**

```
running workflow against run-42.journal
  [plan] REPLAY (not calling fn)
  [search] REPLAY (not calling fn)
  [synthesize] REPLAY (not calling fn)
final answer: [answer to: Synthesize: [answer to: Execute]
```

Check the counter again:

```bash
cat model_calls.count
```

Still **3**. Three model "calls" would-have-been-made; zero of them actually ran. On recovery, Chapter 2's replay semantics returned the recorded results instead of re-invoking the function. The run finished in milliseconds and cost zero tokens.

## Step 7 — Crash mid-run and watch the fast-forward

This is the scenario Chapter 2 is built for: *"If the worker dies after step N completes but before step N+1, recovery replays the workflow... execution resumes at step N+1."*

Start clean, then crash the process after exactly one activity completes. We'll do this by killing the shell command after about 1.5 seconds — long enough for `plan` to finish and record (at ~1.0 s) but not enough for `search` to start and finish.

```bash
rm -f run-42.journal model_calls.count

# Run with a hard timeout that fires after step 1 completes but before step 2 finishes.
timeout --signal=KILL 1.5 python workflow.py || echo "killed mid-run"

cat run-42.journal
cat model_calls.count
```

**Expected output:**

```
running workflow against run-42.journal
  [plan] RUN
  [search] RUN
killed mid-run
{"step_id": "plan", "result": "[answer to: Plan steps for: what is the ca"}
2
```

The journal has **one** completed step (`plan`). The counter is **2** — `plan` completed and recorded, `search` started (incrementing the counter) but was killed before it could record. Now restart:

```bash
python workflow.py
cat model_calls.count
```

**Expected output:**

```
running workflow against run-42.journal
  [plan] REPLAY (not calling fn)
  [search] RUN
  [synthesize] RUN
final answer: [answer to: Synthesize: [answer to: Execute]
```

```
4
```

Read the counter across all three runs: `3` → `2` (killed mid-run after `plan` recorded, `search` incremented then died) → `4` (replay of `plan` didn't bump the counter; `search` ran once, `synthesize` ran once). The second launch did **not** re-pay for `plan`. The workflow fast-forwarded through the recorded step and resumed exactly at the next pending one. Chapter 2: *"The plan model call is never paid for twice."*

> Also notice: `search` ran a second time. It was *interrupted mid-flight* in the first attempt, so its result was never recorded; replay has nothing to return, so the activity retries from scratch. That's the exact gap Step 8 closes.

## Step 8 — Idempotency when an interrupted activity retries

The `search` activity that got killed mid-flight ran its model "call" twice — once in the crashed process, once on restart. For a *pure* model call the only cost is tokens (Chapter 2: *"re-asking the model is usually harmless"*). For a **side-effecting** tool call — "send an email", "charge a card", "write a record" — running twice is a real bug. The engine won't save you; you need an idempotency key.

Add a side-effecting activity. Save as `record_result.py`:

```python
import json
import os
from pathlib import Path


LEDGER = Path("sent.ledger")


def record_result(key: str, payload: str) -> str:
    """Pretend to send an external notification. Idempotent on `key`:
    if `key` is already in the ledger, do nothing; otherwise append it."""
    seen = set()
    if LEDGER.exists():
        with LEDGER.open() as f:
            seen = {json.loads(line)["key"] for line in f if line.strip()}
    if key in seen:
        print(f"    (dedup) key={key!r} already recorded, skipping side effect")
        return "deduped"
    with LEDGER.open("a") as f:
        f.write(json.dumps({"key": key, "payload": payload}) + "\n")
        f.flush()
        os.fsync(f.fileno())
    print(f"    [side effect fired] key={key!r}")
    return "sent"
```

Extend `workflow.py` to call it after `synthesize`:

```python
from record_result import record_result  # at the top

def research_agent(journal: Journal, question: str) -> str:
    plan = activity("plan", journal, call_model, f"Plan steps for: {question}")
    search = activity("search", journal, call_model, f"Execute plan: {plan}")
    answer = activity("synthesize", journal, call_model, f"Synthesize: {search}")
    # Side-effecting step with an idempotency key tied to the run.
    activity("record", journal, record_result, "run-42:record", answer)
    return answer
```

Now simulate the interrupt-and-retry explicitly: the `record_result` function will "run twice" across two process invocations because the first attempt completed its side effect but **crashed before the journal could record it**. To force that, wipe only the journal line (not the ledger) between runs. A shortcut: wrap `record_result` in a flaky one-shot that raises after writing the ledger the first time.

Save as `flaky_record.py`:

```python
from pathlib import Path

from record_result import record_result


FUSE = Path(".flaky.fuse")


def flaky_record(key: str, payload: str) -> str:
    """Runs the real side effect, then raises the first time only.
    Used to simulate an activity that completed its external effect but
    died before the engine could persist its result."""
    result = record_result(key, payload)
    if not FUSE.exists():
        FUSE.write_text("blown")
        raise RuntimeError("worker crashed after side effect, before journal write")
    return result
```

Swap the workflow's `record_result` call for `flaky_record`:

```python
from flaky_record import flaky_record

activity("record", journal, flaky_record, "run-42:record", answer)
```

Clean and run:

```bash
rm -f run-42.journal model_calls.count sent.ledger .flaky.fuse
python workflow.py  # will crash on `record`
echo "---"
python workflow.py  # retries `record`, side effect must fire only once
cat sent.ledger
```

**Expected output:**

```
running workflow against run-42.journal
  [plan] RUN
  [search] RUN
  [synthesize] RUN
  [record] RUN
    [side effect fired] key='run-42:record'
Traceback (most recent call last):
  ...
RuntimeError: worker crashed after side effect, before journal write
---
running workflow against run-42.journal
  [plan] REPLAY (not calling fn)
  [search] REPLAY (not calling fn)
  [synthesize] REPLAY (not calling fn)
  [record] RUN
    (dedup) key='run-42:record' already recorded, skipping side effect
final answer: [answer to: Synthesize: [answer to: Execute]
```

```
{"key": "run-42:record", "payload": "[answer to: Synthesize: [answer to: Execute plan: [answer to: Plan steps for: what is the ca"}
```

Read carefully — this is the Chapter 2 idempotency story in one listing:

- First run: `plan`, `search`, `synthesize`, `record` all ran. The ledger got **one** entry. Then the fuse blew and the process crashed before the journal could record `record`'s result.
- Second run: `plan`/`search`/`synthesize` replay from the journal (no model calls). `record` retries because its result was never journaled — and the retry runs `flaky_record` → `record_result`, which checks the ledger, sees the key, and **short-circuits**. The side effect fired **exactly once**.

Without the idempotency key, the ledger would have **two** identical entries at the end. In production that's a double-charge or a duplicate email, exactly as Chapter 2 warns: *"Replay protects completed activities. It does not protect an activity that was interrupted mid-flight."*

## Step 9 — Feel the "no I/O in the workflow" rule

Edit `workflow.py` and add a line **inside** `research_agent` that reads a clock:

```python
import time, random
def research_agent(journal: Journal, question: str) -> str:
    bias = random.random()                      # <-- non-determinism inside the workflow
    plan = activity("plan", journal, call_model, f"Plan {bias:.3f} for: {question}")
    ...
```

Clean and run twice, inspecting the journal in between:

```bash
rm -f run-42.journal model_calls.count sent.ledger .flaky.fuse
python workflow.py
# Edit one journal line by hand — or just rerun; the key insight is that on a
# replay the workflow would compute a NEW `bias`, call activity("plan", ...) with
# a NEW prompt, and the journal's recorded result (keyed on step_id="plan") would
# bear no relation to the input the workflow is now "re-dispatching" for.
python workflow.py
```

The second run replays `plan`'s recorded result — but the `bias` on this replay is a *different* random number than the one that produced that result. The workflow is now **lying** about what it did. Temporal would reject the entire workflow for being non-deterministic; our tiny journal happily hands back a stale result that doesn't match the "new" prompt. Chapter 2 is strict about this for exactly that reason: *"The engine re-runs it from the start on recovery... No direct I/O, no `random`, no wall-clock `time.now()` inside a workflow."* Remove the bias line before moving on.

## Step 10 — Checkpoint

Without peeking, you can state:

1. What **three** files encode the state of a mid-run durable execution, and which one of them is the "history" the engine replays from. (Journal, persistent counter, side-effect ledger — journal is the history.)
2. The exact scenario in which a completed activity's side effect fires twice on retry — and the one line of code (the idempotency key check) that prevents it.
3. One thing Chapter 2 forbids inside the workflow function, and the concrete failure mode you just saw when you added it.

## Wrap-up

Carry the discipline from this lab into [Exercise 02](../exercises/exercise-02-durable-execution-temporal.md), but let the Temporal SDK carry the mechanics: the `Journal` here is `WorkflowHistory` there, the `@activity` decorator here is `@activity.defn` there, and the "workflow function" here is `@workflow.defn`'s `run` method. The exercise adds things this lab intentionally skipped:

- Timeouts on each activity (`start_to_close_timeout`) and a retry policy with backoff.
- A real worker that can be killed and replaced so a run migrates to another process, not just another invocation.
- A workflow signal / `wait_condition` — the seed of the HITL wait in [Chapter 4](../04-hitl-with-persistence.md).

The idempotency key pattern from Step 8 is the thing you take **unchanged** into every system with retries: Temporal, LangGraph HITL resumes, a job queue, a webhook retry policy. Same discipline, same five lines of code.

### Stretch

- Add a crude retry policy to `activity.py`: on exception, retry up to `n` times with exponential backoff. Rerun Step 8 with `flaky_record` and watch the workflow self-heal without the second-run restart. This is what `RetryPolicy` does in Temporal.
- Make the journal durable to concurrent workers: put a filesystem lock around the `record` call and prove a second process launched against the same `run-42.journal` can't double-record a step.
- Replace the JSON-lines journal with a SQLite file and key activities by `(workflow_id, step_id)`. You've just reinvented the storage layer of a durable workflow engine — minus the parts that matter for correctness at scale.
