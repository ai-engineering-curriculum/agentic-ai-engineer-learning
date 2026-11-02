# exercise-04: Context Management and Session State

**Estimated effort:** 3 hours

## Objective

Extend your exercise-01 harness so it can run a task that's too big for one context window. Add token accounting, clearing of old tool results, compaction by summarization, and a progress file plus feature list on disk so a brand-new session can pick up where the last one stopped. By the end your harness can work across sessions the way long-running agent harnesses do.

## Background

This exercise covers material from:

- [Chapter 5 — Context, Compaction, and Session State](../05-context-compaction-state.md)

It builds directly on [exercise-01](exercise-01-build-a-minimal-harness.md). It doesn't need exercises 02 or 03. The session-state pattern follows the initializer-plus-progress-file approach from Anthropic's long-running harness write-up (see [resources.md](../resources.md)).

## Prerequisites

- Your working exercise-01 harness (or the reference solution).
- Python 3.11+, `pytest`.

## Tasks

### 1. Token accounting

- Track tokens per message. Use the provider's reported usage when available; otherwise estimate (characters ÷ 4 is acceptable for this exercise, but put it behind one function so you can swap in a real counter).
- Expose `context_tokens()` and a configurable `context_limit`.

### 2. Tool-result clearing

- When context passes a first threshold (say 60% of the limit), replace the content of old tool results, oldest first, with a short placeholder (`[cleared: read_file src/app.py, 4,210 tokens]`), keeping the most recent N results intact.
- Never clear the tool call itself. The model must still see that the call happened.
- Keep tool calls and their results paired. A result without its call (or the reverse) breaks most provider APIs.

### 3. Compaction

- When context passes a second threshold (say 85%), ask the model to summarize the conversation so far into a structured note: goal, decisions made, files changed, current state, and next step.
- Replace the history with the system prompt, the summary, and the last few turns.
- Make compaction its own model call with its own small budget, and test it with `ScriptedModel`.

### 4. Session state on disk

- On first run for a task, an initializer step writes `features.json` (a list of features, each with `id`, `description`, and `passes: false`) and an empty `progress.md`.
- Each later session starts by reading `progress.md`, `features.json`, and `git log --oneline -10`, and works on one failing feature.
- The harness, not the model, enforces that `features.json` can only change by flipping `passes` on an existing feature. Any other change to the file is rejected.
- At the end of a session the agent appends to `progress.md` and commits.

### 5. Resume

- Kill a run partway through. Start a fresh process with no in-memory history. Show that it reads the state files and continues with the next failing feature instead of starting over.

## Starter guidance

```python
CLEAR_AT = 0.60
COMPACT_AT = 0.85
KEEP_RECENT_RESULTS = 3

def estimate_tokens(text: str) -> int:
    return max(1, len(text) // 4)

COMPACTION_PROMPT = """Summarize this session for a fresh engineer taking over.
Use these headings: Goal, Decisions, Files changed, Current state, Next step.
Be specific: file paths, function names, failing test names. No narrative."""
```

## Acceptance criteria

You can demonstrate that, in `pytest`:

- Tool-result clearing triggers at the first threshold, keeps the most recent N results, and leaves every tool call paired with a result.
- Compaction triggers at the second threshold and the new history contains the system prompt, the summary, and the recent turns, with context back under the threshold.
- A write to `features.json` that adds, removes, or rewords a feature is rejected; flipping `passes` is accepted.
- A fresh process given only the workspace on disk resumes with the next failing feature.

## Reflection

In `NOTES.md`:

1. What did compaction lose that mattered? How would you tell from a trace?
2. Why does the feature list work better as JSON than as a Markdown checklist?
3. Clearing, compaction, and fresh sessions with state files all trade memory for room. When would you use each?

## Stretch goals

- Add a `memory` tool that lets the model write durable notes to a file it can read in later sessions, and compare it with automatic compaction on the same long task.
- Measure how much clearing alone delays compaction on a read-heavy task.
- Use the provider's server-side context editing or compaction feature, if available, and compare it with your own.
