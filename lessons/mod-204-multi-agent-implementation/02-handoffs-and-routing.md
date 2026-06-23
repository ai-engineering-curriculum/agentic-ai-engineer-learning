# Chapter 2 — Handoffs and Routing

Fan-out spreads a task *across* workers at once. **Handoffs** do the opposite: control moves *sequentially* from one specialist to the next, with only one agent active at a time. This is the right shape when the work is a pipeline or a decision tree, not a parallel sweep.

```
user ──▶ [triage] ──handoff──▶ [billing]
              │
              └────handoff──▶ [technical] ──handoff──▶ [escalation]
```

## Routing vs. orchestration

Two related-but-distinct patterns:

- **Router:** a lightweight classifier (often one cheap model call, no tools) that inspects the request and picks *which* specialist handles it. The router does no work itself; it dispatches. Great for support triage, intent classification, model-tier selection.
- **Handoff:** a specialist agent, mid-task, decides the request now belongs to a *different* specialist and transfers control — passing along the state the next agent needs.

A router is a handoff that happens once, at the front door. Handoffs are routing that can happen repeatedly, from inside the work.

## Implementing a handoff

A handoff is just a **tool** the agent can call. The "tool" doesn't do external work — it signals the runtime to switch the active agent.

```python
HANDOFF_TOOL = {
    "name": "handoff",
    "description": "Transfer the conversation to a more appropriate specialist.",
    "input_schema": {
        "type": "object",
        "properties": {
            "to": {"enum": ["billing", "technical", "escalation"]},
            "summary": {"type": "string",
                        "description": "What the next agent needs to know."},
        },
        "required": ["to", "summary"],
    },
}

async def run_with_handoffs(agents: dict, start: str, user_msg: str, max_hops=5):
    active, context = start, user_msg
    for _ in range(max_hops):
        step = await run_agent(system=agents[active], task=context,
                               extra_tools=[HANDOFF_TOOL])
        if step.handoff:                       # agent called handoff()
            active = step.handoff["to"]
            context = step.handoff["summary"]  # carry forward only what's needed
            continue
        return step.final                      # agent answered; done
    raise RuntimeError("handoff limit exceeded")
```

Two things to notice: the **`summary`** is the entire handoff payload — the next agent starts fresh with just that, not the whole transcript — and there is a **`max_hops`** ceiling.

## Failure modes to design against

- **Handoff loops.** Billing hands to technical, technical hands back to billing, forever. Defend with a hop limit *and* a rule in each system prompt: "Do not hand off to an agent that has already handled this request — answer or escalate instead." Track the visited set and pass it along.
- **Context loss on transfer.** If the `summary` is thin, the next agent re-asks the user questions already answered. Make the schema force the outgoing agent to state the goal, what's been established, and what's still needed.
- **Over-routing.** A router that splits hairs ("is this *really* billing or account?") adds a model call and a failure point for little gain. Keep the route set small and the categories obviously distinct.

## Router vs. orchestrator: choosing

| | Router / Handoff | Orchestrator-Worker |
|---|---|---|
| Agents active at once | One | Many |
| Work shape | Sequential / branching | Parallel / independent |
| Good for | Triage, pipelines, escalation | Research, breadth, fan-out |
| Main risk | Handoff loops, context loss | Cost blow-up, bad decomposition |

Real systems combine them: a router picks a pipeline, and a stage in that pipeline fans out to workers.

## Key takeaways

- Handoffs move control sequentially; implement them as a `handoff` tool whose payload is a **summary**, not a transcript.
- Always bound hops and forbid re-handing-off to a prior agent — loops are the classic failure.
- Routers are one-shot front-door handoffs; keep the category set small and distinct.
