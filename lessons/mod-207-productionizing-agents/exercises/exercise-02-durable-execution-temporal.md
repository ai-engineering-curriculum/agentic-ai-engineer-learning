# exercise-02: Durable Execution with Temporal

**Estimated effort:** 3 hours

## Objective

Convert a multi-step agent run into a durable Temporal workflow so that killing the worker mid-run resumes instead of restarting, completed model calls are never re-paid, and a side-effecting tool fires exactly once. By the end you'll have felt replay first-hand and made an activity idempotent under retry.

## Background

This exercise covers material from:

- [Chapter 2 — Durable Execution and Resumption](../02-durable-execution.md)

Use the agent from [exercise-01](exercise-01-agent-api-deployment.md) or your mod-201 loop. The point is the durability layer, so a small agent with three or four steps is plenty.

## Prerequisites

- A multi-step agent run (at least three sequential steps, one with an external side effect).
- The Temporal Python SDK (`temporalio`) and a local Temporal dev server running.
- API key in an environment variable; small spend cap.

## Tasks

### 1. Split into workflow and activities

- Move each side-effecting step (each model call, each tool call) into an `@activity.defn` function.
- Put the orchestration — the order of steps and the branching — into an `@workflow.defn` workflow. Keep the workflow deterministic: no direct I/O, no `random`, no wall-clock reads inside it.

### 2. Run it end to end

- Start a worker that hosts the workflow and activities.
- Trigger the workflow with a real question and confirm it completes and returns the agent's answer.

### 3. Prove replay survives a crash

- Add a log line at the start of each activity and each workflow entry.
- Kill the worker **after** at least one activity has completed but before the run finishes. Restart it.
- Show from the logs that completed activities are **not** re-executed (their model calls don't run again) and the run finishes from where it stopped.

### 4. Idempotent side effect

- Make one activity perform an external side effect (e.g., "record this result" to a file or table, or a stubbed "send notification").
- Give it an idempotency key derived from the workflow id + step so that an interrupted-then-retried activity fires the effect exactly once.
- Force a retry on that activity (raise once, then succeed) and prove the side effect happened a single time.

### 5. Retry policy

- Configure a retry policy on a flaky activity (backoff, max attempts) and demonstrate it recovers from a transient error without restarting the whole run.

## Starter guidance

```python
from datetime import timedelta
from temporalio import workflow, activity

@activity.defn
async def plan_step(question: str) -> str:
    return await run_model(f"Plan: {question}")  # pure model call, safe to replay

@activity.defn
async def record_result(key: str, payload: str) -> None:
    # External side effect — must be idempotent under retry.
    if not already_recorded(key):
        write_record(key, payload)

@workflow.defn
class DurableAgent:
    @workflow.run
    async def run(self, question: str) -> str:
        plan = await workflow.execute_activity(
            plan_step, question,
            start_to_close_timeout=timedelta(minutes=2),
        )
        # ... more steps ...
        key = f"{workflow.info().workflow_id}:record"
        await workflow.execute_activity(
            record_result, args=[key, plan],
            start_to_close_timeout=timedelta(seconds=30),
        )
        return plan
```

You do **not** need the FastAPI layer here, though wiring the workflow behind the `POST /agent/jobs` endpoint from exercise-01 is a fair stretch goal.

## Acceptance criteria

You can demonstrate that:

- The agent run is split into a deterministic workflow and idempotent activities, with no I/O inside the workflow.
- Killing and restarting the worker mid-run resumes the run; completed activities' model calls do **not** run a second time (shown in logs).
- The side-effecting activity, forced to retry, performs its external effect exactly once.
- A flaky activity recovers via its retry policy without restarting the whole workflow.

## Reflection

In `NOTES.md`:

1. Which of your activities were naturally safe to replay, and which needed an idempotency key? What made the difference?
2. What broke (or would break) if you left an `import random` or a `datetime.now()` inside the workflow function?
3. For your agent, where's the line — at what run length / cost does durable execution start to earn its complexity?

## Stretch goals

- Add a `workflow.wait_condition` or signal so the workflow can pause for an external event — the seed of the human-in-the-loop wait in [exercise-04](exercise-04-hitl-with-persistence.md).
- Expose the workflow behind the exercise-01 `POST /agent/jobs` endpoint so the API returns a workflow id the client can poll.
- Add a heartbeat to a long activity and demonstrate Temporal detecting a stalled worker.
