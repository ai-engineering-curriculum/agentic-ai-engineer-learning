# exercise-03: Caching and Routing

**Estimated effort:** 3 hours

## Objective

Cut your agent's cost and latency two ways: add prompt caching to the stable prefix and add a model router that sends each turn to the smallest capable model, then **measure** both wins against a baseline. By the end you'll be able to prove a cost/latency improvement from the usage numbers without losing quality on your eval set.

## Background

This exercise covers material from:

- [Chapter 3 — Prompt Caching and Model Routing](../03-caching-and-routing.md)

Use the Anthropic API for the caching half (the `cache_control` mechanism and the `usage` cache fields are concrete there). Bring a small eval set — even 10–20 representative turns with expected outcomes — from [mod-205](../../mod-205-evaluation-observability/README.md) or write one.

## Prerequisites

- An agent whose prompt has a large, stable head (system prompt, tool definitions, and/or retrieved context) and a small changing tail.
- The `anthropic` Python SDK and an API key in an environment variable; small spend cap.
- A small eval set (~10–20 turns) with a pass/fail or scored check per turn.

## Tasks

### 1. Baseline

- Run your eval set with no caching and a single fixed model. Record, per turn: input tokens, output tokens, latency, and the eval result.
- Compute totals: total tokens, total cost, mean and p95 latency, pass rate.

### 2. Add prompt caching

- Mark the stable prefix (system + tools + context) with `cache_control: {"type": "ephemeral"}`, ordered stable-first, volatile-last.
- Run a burst of related turns and read `cache_creation_input_tokens` and `cache_read_input_tokens` from `usage`.
- Show the first call writes the cache and subsequent calls read it (read tokens > 0).

### 3. Add a model router

- Write `choose_model(turn)` that routes by task kind and/or cheap input heuristics (context length, needs-reasoning flag) to a small / mid / large model.
- Route the eval set and record the model chosen per turn.

### 4. Add an escalation path

- Define a failure check on the cheap-model output (low confidence, invalid structure, or a self-reported uncertainty).
- On failure, retry the turn on the larger model. Record how often escalation fires.

### 5. Measure and compare

- Produce a table comparing baseline vs. cached vs. cached+routed: total cost, mean/p95 latency, and **pass rate**.
- Confirm the pass rate did not drop materially. A cost win that lowers quality is a regression — say so if it happened.

## Starter guidance

```python
import anthropic

client = anthropic.Anthropic()

def cached_call(model: str, system_prefix: str, user_turn: str):
    return client.messages.create(
        model=model,
        max_tokens=1024,
        system=[
            {"type": "text", "text": system_prefix,
             "cache_control": {"type": "ephemeral"}},
        ],
        messages=[{"role": "user", "content": user_turn}],
    )

def choose_model(turn: dict) -> str:
    if turn["kind"] in {"classify", "extract", "format"}:
        return "claude-3-5-haiku-latest"
    if turn.get("needs_reasoning") or turn.get("context_tokens", 0) > 50_000:
        return "claude-opus-4-1"
    return "claude-sonnet-4-0"

def run_turn(turn: dict, system_prefix: str):
    model = choose_model(turn)
    resp = cached_call(model, system_prefix, turn["input"])
    if failed_check(resp) and model != "claude-opus-4-1":
        resp = cached_call("claude-opus-4-1", system_prefix, turn["input"])  # escalate
    return resp
```

You do **not** need durability or HITL here — focus on the cost/latency levers and the measurement.

## Acceptance criteria

You can demonstrate that:

- Caching is verified by the usage fields: `cache_read_input_tokens` is non-zero on calls after the first.
- The router assigns models by difficulty, and you can show which turns went to which model.
- The escalation path catches at least one cheap-model failure and retries it on the larger model.
- Your comparison table shows a real cost and/or latency reduction with **no material drop in pass rate**.

## Reflection

In `NOTES.md`:

1. What fraction of your savings came from caching vs. routing? Which lever mattered more for your agent, and why?
2. Where did the cache *fail* to help (one changed token early, a prefix called once)? How did you reorder to fix it?
3. Did aggressive routing ever drop quality? What's your escalation trigger, and how did you tune it?

## Stretch goals

- Replace the static router with a tiny model-based difficulty classifier and compare its routing decisions and net cost against the rule-based one.
- Add the Batches API for any non-time-sensitive bulk turns and fold the 50% savings into your comparison.
- Track cache hit rate over a longer multi-turn session and find where the ephemeral cache expires.
