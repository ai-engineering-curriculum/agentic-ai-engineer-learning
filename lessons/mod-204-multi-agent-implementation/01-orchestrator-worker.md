# Chapter 1 — The Orchestrator-Worker Pattern

The orchestrator-worker pattern is the workhorse of multi-agent systems: one **orchestrator** (or "lead") agent decomposes a task into independent sub-tasks, dispatches each to a **worker** agent, and then **synthesizes** the workers' results into a final answer.

```
                ┌─────────────┐
   task ───────▶│ orchestrator│  decompose
                └──────┬──────┘
            ┌──────────┼──────────┐
            ▼          ▼          ▼
        ┌───────┐  ┌───────┐  ┌───────┐
        │worker │  │worker │  │worker │   (run concurrently)
        └───┬───┘  └───┬───┘  └───┬───┘
            └──────────┼──────────┘
                       ▼
                ┌─────────────┐
                │ orchestrator│  synthesize ──▶ result
                └─────────────┘
```

## When it beats a single agent

Reach for orchestrator-worker when:

- **The work decomposes into independent sub-tasks.** "Research these 5 companies" → one worker per company. If sub-tasks depend on each other in sequence, you want handoffs ([Chapter 2](02-handoffs-and-routing.md)), not fan-out.
- **Breadth would blow the context window.** Five workers each reading 20 documents keep 20 docs in *their* context; the orchestrator only ever sees five summaries ([Chapter 4](04-subagent-isolation.md)).
- **Sub-tasks benefit from specialization.** A `code-writer` worker and a `test-writer` worker can carry different system prompts and tools.

It is *not* free: every worker is extra latency and tokens, and the orchestrator's decomposition can be wrong. For a task a single agent handles in a few tool calls, a single agent is cheaper and more reliable.

## The three phases

**1. Decompose.** The orchestrator turns the task into a list of concrete worker assignments. Make it return *structured* output — a list of `{worker_role, instruction}` — not prose you have to parse. This is where most failures originate: vague or overlapping assignments produce redundant or contradictory work.

**2. Fan out.** Run the workers **concurrently**. Each worker is its own agent loop (reason-act from [mod-201](../mod-201-agent-fundamentals/README.md)) with only its assignment in context — not the whole conversation.

```python
import asyncio

async def run_workers(assignments: list[dict]) -> list[dict]:
    async def one(a):
        result = await run_agent(
            system=WORKER_SYSTEM[a["worker_role"]],
            task=a["instruction"],
        )
        return {"assignment": a, "result": result}
    return await asyncio.gather(*(one(a) for a in assignments))
```

**3. Synthesize.** The orchestrator receives the workers' *distilled* results and writes the final answer. Give it the explicit synthesis instruction ("combine these findings; flag disagreements; do not invent facts not present in the results").

## Design rules that save you

- **Bound the fan-out.** Have the orchestrator emit *at most N* assignments. An unbounded decomposition that spawns 200 workers is a runaway cost incident.
- **Make assignments self-contained.** A worker can't see the conversation; everything it needs goes in its instruction. If a worker keeps asking for context it doesn't have, your decomposition is leaking assumptions.
- **Workers return results, not transcripts.** A worker's job is to hand back a clean answer, not its scratch work — see [Chapter 4](04-subagent-isolation.md).
- **Handle partial failure.** One worker erroring shouldn't sink the run. Collect successes, tell the orchestrator which assignments failed, and let it synthesize from what came back (or retry that one).

## Key takeaways

- Orchestrator-worker = decompose → fan out concurrently → synthesize.
- Use it for **independent, broad** work; use handoffs for **sequential** work.
- Structured assignments, bounded fan-out, self-contained instructions, and distilled returns are what separate a working system from an expensive mess.
