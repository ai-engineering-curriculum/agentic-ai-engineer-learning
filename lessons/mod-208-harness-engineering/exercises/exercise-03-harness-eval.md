# exercise-03: Evaluate a Harness Change

**Estimated effort:** 3 hours

## Objective

Build a small eval runner that answers "did this harness change help?" with numbers instead of a feeling. It runs a headless coding agent against a fixed task set, several trials per task, under two harness configurations, scores each trial with the task's own check command, and reports pass rate, cost, and turns side by side. By the end you'll have a tool you can rerun every time you edit `AGENTS.md` or add a hook.

## Background

This exercise covers material from:

- [Chapter 4 — Sensors: Feedback Loops and Harness Evals](../04-sensors-feedback-and-evals.md)

You built trajectory evals and CI regression suites in [mod-205](../../mod-205-evaluation-observability/README.md). This exercise applies the same idea one level up: the thing under test is the harness, and the model stays fixed.

## Prerequisites

- Python 3.11+, `pytest`.
- For live runs: Claude Code (`claude -p ... --output-format json`) or Codex CLI (`codex exec --json`), plus a spend cap. Develop and test against the fake runner first.
- The repo and tasks from [exercise-01](exercise-01-harness-a-repository.md), or any repo with three or more checkable tasks.

## Tasks

### 1. Task format

- Define tasks in `tasks.yaml` (or JSON). Each task has: `id`, `prompt`, `setup` (shell command to reset the repo to the task's starting state, e.g. `git checkout <sha>`), and `check` (shell command whose exit code decides pass or fail).
- Validate the file on load. Reject tasks missing a field with a clear error, not a `KeyError` deep in the run.

### 2. Runner interface

- Define a `Runner` protocol with one method: `run(prompt, workdir, harness) -> TrialResult`, where `TrialResult` holds `turns`, `cost_usd`, `duration_s`, and `error` (or `None`).
- Implement `FakeRunner`: deterministic, no network, driven by a seed so tests are repeatable. It should be able to simulate a harness that helps (higher pass probability) and one that doesn't.
- Implement `ClaudeCodeRunner` (or `CodexRunner`): runs the CLI as a subprocess in `workdir`, parses its JSON output for turns and cost, and enforces a wall-clock timeout.

### 3. Harness configurations

- A harness is a directory of files copied into the workdir before each trial (for example `harnesses/baseline/` with nothing in it, and `harnesses/v2/` with `AGENTS.md`, `.claude/settings.json`, and hooks).
- Each trial gets a clean copy of the repo. Trials must not see each other's changes.

### 4. Run and score

- For each harness × task × trial (default 5 trials): reset, apply the harness, run the agent, then run `check` and record pass or fail.
- Write every trial as one JSON line to `results.jsonl` so a crashed run can be inspected and resumed.

### 5. Report

- Print a table per harness: pass rate overall and per task, mean cost, mean turns, error count.
- For the pass-rate difference between the two harnesses, report a simple uncertainty: a bootstrap 95% interval over trials. Say "no clear difference" when the interval includes zero.

## Starter guidance

```python
from dataclasses import dataclass
from pathlib import Path
from typing import Protocol

@dataclass(frozen=True)
class Task:
    id: str
    prompt: str
    setup: str
    check: str

@dataclass(frozen=True)
class TrialResult:
    turns: int
    cost_usd: float
    duration_s: float
    error: str | None = None

class Runner(Protocol):
    def run(self, prompt: str, workdir: Path, harness: Path) -> TrialResult: ...
```

Headless invocation shapes to start from (confirm flags against your installed version):

```bash
claude -p "$PROMPT" --output-format json --max-budget-usd 1.00
codex exec --json "$PROMPT"
```

## Acceptance criteria

You can demonstrate that:

- `pytest` passes using only `FakeRunner`: task validation, trial isolation, scoring by exit code, JSONL output, and report math.
- With `FakeRunner` configured so harness B is better, the report shows B ahead with an interval that excludes zero; with both configured equal, it reports no clear difference.
- One live run of at least 3 tasks × 3 trials × 2 harnesses completes, and `results.jsonl` has one line per trial.
- A trial that times out or crashes is recorded as an error, not dropped, and doesn't stop the run.

## Reflection

In `NOTES.md`:

1. What did your live run show about the harness from exercise-01? Was the difference outside the noise?
2. Your `check` command is the only judge. Name a way an agent could pass `check` without doing the task, and how you'd close that gap.
3. Cost per trial times trials times tasks times harnesses adds up. How would you size an eval you can afford to run on every harness change?

## Stretch goals

- Run trials in parallel with a concurrency cap, keeping each in its own git worktree.
- Add a second scorer: an LLM judge that reads the diff and rates whether the change matches the task intent, and compare it with `check`.
- Track results over time and fail CI when pass rate drops by more than a set margin.
