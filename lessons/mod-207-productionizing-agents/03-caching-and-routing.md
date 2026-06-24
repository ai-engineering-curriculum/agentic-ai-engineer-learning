# Chapter 3 — Prompt Caching and Model Routing

Two levers cut an agent's cost and latency without touching its logic: **caching** the part of the prompt that doesn't change, and **routing** each turn to the cheapest model that can handle it. Both are measured, not assumed — you read the win off the usage numbers.

## Prompt caching: pay once for the stable prefix

An agent's prompt has a large, stable head — system instructions, tool definitions, retrieved context, few-shot examples — followed by a small, changing tail (the latest user turn). Without caching you re-send and re-process that whole head on every call. Prompt caching lets the provider store the processed prefix and reuse it: cached tokens are read at a steep discount and skip recomputation, so you save **both** money and time-to-first-token.

With the Anthropic API you mark the end of a cacheable block with `cache_control`. Everything *before* the breakpoint becomes the cached prefix.

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": large_system_prompt_and_context,
            "cache_control": {"type": "ephemeral"},
        }
    ],
    messages=[{"role": "user", "content": "The changing question"}],
)

print("cache writes:", message.usage.cache_creation_input_tokens)
print("cache reads: ", message.usage.cache_read_input_tokens)
```

Rules that make caching actually pay off:

- **Order matters.** Put stable content first, volatile content last. The cache matches a *prefix*; one changed token early invalidates everything after it.
- **The first call writes, later calls read.** A cache write costs slightly more than a normal token; the savings come on reuse. Caching a prefix you call once is a net loss.
- **The cache is short-lived.** The ephemeral cache expires after a few minutes of inactivity, refreshed on each hit. It's built for a burst of related calls — a single agent run, a multi-turn conversation — not for cross-day reuse.
- **Verify with the usage fields.** `cache_read_input_tokens` rising on subsequent calls is your proof the cache is working. If it stays zero, your prefix is changing or too small to cache.

## Model routing: cheap by default, expensive on demand

Not every turn needs your most capable model. A classification, an extraction, a simple tool selection, or a short factual reply runs fine — and far cheaper and faster — on a small model; deep reasoning and synthesis need a large one. Routing sends each request to the smallest model that will succeed.

```python
def choose_model(turn: dict) -> str:
    if turn["kind"] in {"classify", "extract", "format"}:
        return "claude-3-5-haiku-latest"
    if turn["needs_reasoning"] or turn["context_tokens"] > 50_000:
        return "claude-opus-4-1"
    return "claude-sonnet-4-0"
```

Routing strategies, simplest first:

- **Static rules.** Route by task type or a difficulty flag your orchestrator already knows. Cheapest to build, surprisingly effective, fully predictable.
- **Heuristics on the input.** Context length, presence of code, explicit user "think hard" — cheap signals that correlate with difficulty.
- **A model-based router.** A tiny fast model classifies difficulty before the real call. More accurate, but adds a call and a failure point — earn it with data.

The trap is silent quality loss: route too aggressively and a cheap model botches a hard turn. Always pair routing with an **escalation path** — if the cheap model's output fails a check (low confidence, invalid structure, a self-reported "I'm not sure"), retry on the larger model. Measure the blend against your eval set ([mod-205](../mod-205-evaluation-observability/README.md)); a cost win that tanks quality isn't a win.

## Stacking them

Caching and routing compose. Cache the stable prefix so every model in the route reads it cheaply, and route the volatile tail to the right-sized model. Caching attacks the *cost per call*; routing attacks *which call you make*. Together they often cut an agent's spend by a large multiple — but only if you keep an eye on the usage numbers and the eval scores while you tune.

## Key takeaways

- Cache the stable prompt prefix (system, tools, context); put volatile content last so it doesn't invalidate the cache.
- Caching pays off on **reuse** within a short window; verify with `cache_read_input_tokens`, and don't cache prefixes you call once.
- Route each turn to the smallest model that can handle it, using static rules or cheap input heuristics first.
- Always pair routing with an escalation path and measure the blend against an eval set — a cost win that drops quality is a regression.
