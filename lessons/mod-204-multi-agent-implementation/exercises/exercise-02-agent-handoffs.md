# exercise-02: Agent Handoffs

**Estimated effort:** 3 hours

## Objective

Build a sequential multi-agent system where control **hands off** between specialist agents instead of fanning out. You'll implement a support-desk flow — a triage router that dispatches to specialists, and specialists that can hand off to each other — with the loop protections that keep handoffs from cycling forever or losing context.

## Background

This exercise covers material from:

- [Chapter 2 — Handoffs and Routing](../02-handoffs-and-routing.md)
- [Chapter 3 — Inter-Agent Communication: A2A and MCP](../03-inter-agent-communication.md) (the handoff payload is a typed message)

## Prerequisites

- The agent loop from mod-201 with tool calling.
- API key in env; small spend cap.

## Tasks

### 1. Specialists

- Define three specialist agents, each with its own system prompt: `billing`, `technical`, and `escalation`. Give them deliberately narrow scopes so realistic requests sometimes land on the wrong one.

### 2. The router (front-door handoff)

- Write a lightweight router: one cheap model call, no tools, that classifies an incoming request and picks the starting specialist. The router does no work itself.

### 3. The handoff tool

- Implement `handoff` as a tool any specialist can call, with a typed payload: `{to, summary, visited}`. The `summary` is everything the next agent needs (goal, what's established, what's still open). The next agent starts with **only the summary**, not the transcript.

### 4. Loop protection

- Enforce a **hop limit** (`max_hops = 5`).
- Track a **visited set** and pass it in the payload; forbid handing off to an agent already in it. Each specialist's system prompt must say: "Do not hand off to an agent that has handled this request — answer or escalate."

### 5. Demonstrate

- Run three requests: one the router places correctly and the specialist resolves; one the router **mis-routes** that gets handed off to the right specialist; and one crafted to *try* to loop — show your protections stop it and it ends at `escalation`.

## Starter guidance

```python
HANDOFF_TOOL = {
    "name": "handoff",
    "description": "Transfer to a more appropriate specialist.",
    "input_schema": {
        "type": "object",
        "properties": {
            "to": {"enum": ["billing", "technical", "escalation"]},
            "summary": {"type": "string"},
        },
        "required": ["to", "summary"],
    },
}

async def route(user_msg: str) -> str:
    raise NotImplementedError  # cheap classifier → starting specialist

async def run(agents, user_msg, max_hops=5):
    active = await route(user_msg)
    context, visited = user_msg, {active}
    for _ in range(max_hops):
        step = await run_agent(system=agents[active], task=context,
                               extra_tools=[HANDOFF_TOOL], forbid=visited)
        if not step.handoff:
            return step.final
        active = step.handoff["to"]
        context = step.handoff["summary"]
        visited.add(active)
    return "escalated: handoff limit reached"
```

## Acceptance criteria

You can demonstrate that:

- The router selects a starting specialist with a single cheap call and no tools.
- A handoff transfers control carrying a **summary** — the receiving agent never sees the prior transcript, yet doesn't re-ask answered questions.
- A mis-routed request reaches the correct specialist via handoff.
- A request engineered to loop is stopped by the visited-set rule and/or hop limit and terminates cleanly (at `escalation`).

## Reflection

In `NOTES.md`:

1. What went into your handoff `summary` schema? What broke when it was too thin?
2. Compare this to fanning out (exercise-01): which problems suit handoffs, which suit fan-out?
3. How would you detect handoff loops in production *before* they hit the hop cap?

## Stretch goals

- Replace the freeform `summary` string with a typed `AgentMessage` (intent/task_id/payload) from Chapter 3 and validate it on receipt.
- Add a `human` handoff target that pauses the loop and waits for input.
- Log a trace of the handoff path per request (`triage → technical → escalation`) for later observability work in mod-205.
