# Chapter 2 — Durable Execution and Resumption

A long agent run is a sequence of expensive, side-effecting steps: call the model, hit a tool, call the model again, write to a database, send an email. If the process crashes at step 7 of 10, restarting from step 1 means re-paying for steps 1–6 *and* re-running their side effects — duplicate emails, duplicate charges. Durable execution solves this: completed steps are recorded, and on recovery the engine **replays** them from the log instead of re-executing.

## The core idea: workflows and activities

A durable workflow engine like Temporal splits your code into two kinds of functions:

- **Workflow** — the orchestration logic. It must be **deterministic**: given the same history, it makes the same decisions. The engine re-runs it from the start on recovery, feeding it the recorded results of past steps, so it fast-forwards to where it left off. No direct I/O, no `random`, no wall-clock `time.now()` inside a workflow.
- **Activity** — a single side-effecting step (a model call, a tool call, a DB write). The engine runs each activity *once*, records its result durably, and on replay hands back the recorded result instead of running it again.

```python
from temporalio import workflow, activity

@activity.defn
async def call_model(prompt: str) -> str:
    # The actual side effect. Runs once; result is persisted.
    return await model_client.complete(prompt)

@workflow.defn
class ResearchAgent:
    @workflow.run
    async def run(self, question: str) -> str:
        plan = await workflow.execute_activity(
            call_model, f"Plan steps for: {question}",
            start_to_close_timeout=timedelta(minutes=2),
        )
        findings = await workflow.execute_activity(
            call_model, f"Execute plan: {plan}",
            start_to_close_timeout=timedelta(minutes=5),
        )
        return findings
```

If the worker dies after `plan` completes but before `findings`, recovery replays the workflow: `call_model` for the plan returns its *recorded* result instantly, and execution resumes at `findings`. The plan model call is never paid for twice.

## Idempotency is on you

Replay protects *completed* activities. It does **not** protect an activity that was interrupted mid-flight — that one is retried, so it can run twice. Any activity with an external side effect must be **idempotent** or guarded by an idempotency key:

- Sending an email or charging a card: pass a stable idempotency key the downstream system deduplicates on.
- Appending to a database: use an upsert keyed on the workflow id + step, not a blind insert.
- A pure model call with no side effect is naturally safe to retry.

> The agent's own LLM calls are the easy case — re-asking the model is usually harmless (just costs tokens). The dangerous activities are the *tools* that change the world. Make those idempotent first.

## What this buys a production agent

- **Survives deploys and crashes.** A worker can be killed and replaced mid-run; the workflow resumes on another worker.
- **Built-in retries with backoff.** Configure per-activity retry policies; a flaky tool call retries without restarting the run.
- **Long waits are free.** A workflow can sleep for hours or wait for an external signal (a human approval — [Chapter 4](04-hitl-with-persistence.md)) without holding a process open.
- **An execution history.** You get a durable, inspectable log of every step — invaluable for debugging an agent that misbehaved in production ([mod-205](../mod-205-evaluation-observability/README.md)).

The cost is structure: you split your agent into a deterministic orchestration layer and a set of idempotent activities, and you run workers and a Temporal service. For a 10-second single-turn agent that's overkill. For a multi-minute, multi-tool run you can't afford to restart, it's the difference between robust and fragile.

## Key takeaways

- Durable execution records completed steps and **replays** them on recovery instead of re-running them.
- Split the agent into a deterministic **workflow** (orchestration, no I/O) and idempotent **activities** (the side-effecting steps).
- Replay protects completed activities; interrupted ones retry, so side-effecting tools must be idempotent or keyed.
- The payoff: survives crashes/deploys, free long waits, automatic retries, and an inspectable history — at the cost of structuring the agent this way and running the engine.
