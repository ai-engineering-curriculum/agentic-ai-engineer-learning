# Chapter 4 — Sensors: Feedback Loops and Harness Evals

Sensors tell the agent, and you, whether the work is right. Without them, an agent stops when it *believes* it's done, and that belief is often wrong. This chapter covers building feedback the agent can act on, structuring long tasks so progress survives across sessions, and measuring whether a harness change actually helped.

## Give the agent a way to check its work

The most valuable single thing in a coding harness is a fast, reliable way for the agent to check its own work. With one, the agent can iterate: change, check, read the failure, fix, check again. Without one, every mistake reaches you.

- **One check command.** `make check` (or `just check`, or `scripts/check.sh`) that runs format check, lint, type check, and tests, and exits non-zero on any failure. Name it in the instruction file as the definition of done. One command is easier to follow than five.
- **Fast.** If the full suite takes 20 minutes, the agent will skip it or run it once at the end. Give it a fast path (the tests for the files it touched) plus the full suite before finishing.
- **Reliable.** Flaky tests teach the agent that failures can be ignored. Fix or quarantine them.
- **End to end where it matters.** Unit tests can pass while the feature is broken. For user-facing work, give the agent a way to check like a user: a browser automation tool, a screenshot, a real request against a running server. Anthropic's long-running harness write-up found that agents marked features done far more reliably once they tested them end to end in a browser.

## Error messages written for a model

A sensor is only as useful as its output. Linters and checks written for humans often say *what* is wrong but not *how to fix it*, and a human fills the gap from experience. An agent fills the gap by guessing.

Custom checks are cheap to write, and you can make their messages instructions:

```text
# Before
error: import of 'api.handlers' from 'core.pricing' not allowed

# After
core/pricing.py imports api.handlers. core/ must not depend on api/ (core is pure logic).
Move the shared type to core/types.py and import it from there in both places.
See docs/architecture.md#layers.
```

The second message turns a sensor into a guide for the next attempt. Structural tests like this one, which enforce architecture rules (layer boundaries, naming, file size limits, no new dependencies without approval), are among the highest-value harness additions. They turn rules that used to live in code review into checks the agent hits right away.

## Long-running work across sessions

Some tasks don't fit in one context window. Even with compaction ([mod-209, Chapter 5](../mod-209-building-custom-harnesses/05-context-compaction-state.md)), a long session drifts: the agent loses track of what's done, starts several things at once, or declares victory early. Anthropic's write-up on harnesses for long-running agents describes a structure that works well:

1. **An initializer session** sets up the environment once: an `init.sh` that starts the app, a **feature list** of end-to-end behaviors (all initially failing), an empty **progress file**, and a first git commit.
2. **Every later session** starts the same way: read the progress file, the feature list, and the recent git log; run `init.sh`; run a basic end-to-end check to confirm nothing is broken; then pick **one** failing feature.
3. **The session ends** by testing that feature end to end, marking it passing only if it passes, appending to the progress file, and committing.

A few details make this work:

- **The feature list is JSON, not Markdown.** The write-up reports that models were less likely to rewrite or delete entries in a JSON file than in a Markdown checklist. Agents may only flip each feature's `passes` field. A harness can enforce that with a hook, which is what [mod-209's exercise-04](../mod-209-building-custom-harnesses/exercises/exercise-04-context-and-session-state.md) has you do.
- **One feature per session** stops the agent from starting five things and finishing none before context runs out.
- **Git is the checkpoint.** Descriptive commits let a later session see what happened and revert a bad change.

The same pattern works for human-supervised work. A `PLAN.md` or `progress.md` the agent updates as it goes makes it cheap to start a fresh session when context gets cluttered, instead of pushing on in a degraded one.

## Evaluate the harness, not just the agent

Every harness change is a hypothesis: "adding this line will stop the agent from editing generated files." Most teams test it by trying one task once and deciding it feels better. That isn't enough. Agent runs are noisy. The same task with the same harness can pass on one run and fail on the next.

A harness eval keeps the model fixed and varies the harness:

- **A fixed task set.** Real tasks from your repo, each with a starting state (a commit) and a check command that decides pass or fail. Ten good tasks beat a hundred vague ones.
- **Repeated trials.** Run each task several times per harness. Three to five trials per task is a minimum for spotting anything but large effects.
- **Clean isolation.** Each trial starts from a fresh copy of the repo. Leftovers from one trial must not help or hurt the next.
- **Several metrics.** Pass rate is the headline, but track cost, turns, and wall-clock time too. A harness that passes the same tasks in half the turns is better.
- **Honest uncertainty.** Report an interval on the difference, not just two averages. If 9/15 vs. 11/15 has an interval that includes zero, say "no clear difference".

Coding agents run headless, so this is scriptable: `claude -p "<task>" --output-format json` and `codex exec --json "<task>"` both print a machine-readable result with turn and cost data. Exercise-03 has you build the runner.

Watch for **reward hacking**: an agent that makes `check` pass without doing the task, for example by deleting the failing test, special-casing the test input, or weakening an assertion. Defenses include tests the agent can't edit (deny writes to them, or keep held-out tests outside the workspace), reviewing a sample of diffs by hand, and an inferential sensor that compares the diff to the task's intent.

## Key takeaways

- A fast, reliable, single check command is the most valuable sensor. Make it the definition of done.
- Write check and linter messages that tell the agent how to fix the problem. Structural tests turn review comments into instant feedback.
- For long work: an initializer session, a JSON feature list, a progress file, one feature per session, and git as the checkpoint.
- Evaluate harness changes with a fixed task set, repeated isolated trials, several metrics, and an honest interval.
- Guard against reward hacking with protected or held-out tests and diff review.
