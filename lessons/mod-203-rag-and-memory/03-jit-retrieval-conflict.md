# Chapter 3 — Just-in-Time Retrieval and Conflict Resolution

The naive way to use RAG and memory is **eager**: on every turn, retrieve a pile of chunks and memories and stuff them into the prompt. It works in a demo and rots in production. Eager retrieval burns tokens on context the turn doesn't need, buries the relevant fact among irrelevant ones (the "needle in a haystack" problem), and slows every turn with a retrieval call that often returns nothing useful. The fix is **just-in-time (JIT) retrieval**: the agent retrieves only when the current step actually needs external information, and only what that step needs.

## Eager vs. just-in-time

```text
EAGER:  every turn ──▶ retrieve everything ──▶ huge prompt ──▶ model
                       (most of it unused, signal buried)

JIT:    turn ──▶ model decides: do I need to look something up?
                   │
                   ├─ no  ──▶ answer directly
                   └─ yes ──▶ search_docs(specific query) ──▶ inject ──▶ answer
```

JIT falls straight out of treating retrieval as a **tool** ([Chapter 1](01-rag-fundamentals.md)). The agent's reason-act loop already decides whether to call a tool; retrieval is one more. You give it `search_docs` and `recall_memory` tools and a system prompt that says *when* to reach for them ("look up specifics you're unsure of; don't retrieve for things you already know"). The model now pulls context on demand instead of you pushing it on every turn.

```python
TOOLS = [
    {"name": "search_docs",
     "description": "Search the knowledge base for specific facts you are "
                    "unsure of. Use a precise query. Don't search for things "
                    "already in the conversation.",
     "input_schema": {"type": "object",
                      "properties": {"query": {"type": "string"}},
                      "required": ["query"]}},
    {"name": "recall_memory",
     "description": "Retrieve durable facts you've stored about this user.",
     "input_schema": {"type": "object",
                      "properties": {"cue": {"type": "string"}},
                      "required": ["cue"]}},
]
# the agent loop runs these only when the model calls them — that's JIT.
```

The trade-off is real: JIT adds a model round-trip when retrieval *is* needed (decide → search → answer instead of search-then-answer). For a workload where almost every turn needs the same documents, eager pre-loading is simpler and cheaper. JIT wins when retrieval is *sometimes* needed and the corpus is large — which is most agent workloads.

## When retrieved facts disagree

Retrieve from a real corpus and a real memory store and you will eventually pull back **contradictory** facts: an old doc says the API rate limit is 100/s, a newer one says 1000/s; the user said "I'm on the free tier" last month and "I upgraded to pro" today. If you dump both into the prompt and hope, the model picks arbitrarily. **Conflict resolution** is the deliberate policy for which fact wins. Three signals, roughly in priority order:

1. **Recency.** Newer usually beats older — the user upgraded, the doc was revised. This is why every memory and chunk needs a **timestamp** in its metadata. Recency is the default tiebreaker, but not absolute: a newer low-quality source shouldn't beat an older authoritative one.
2. **Source authority.** A fact from official docs outranks one scraped from a forum; a fact the user stated directly outranks one the agent inferred. Tag each memory and chunk with a **source** and rank sources explicitly.
3. **Confidence.** If a memory was written with a confidence score (or you can estimate one), prefer the higher-confidence fact. Low-confidence inferences should yield to high-confidence statements.

```python
def resolve(conflicting: list[dict]) -> dict:
    # each item: {"fact", "timestamp", "source", "confidence"}
    return max(
        conflicting,
        key=lambda m: (
            SOURCE_RANK[m["source"]],   # authority first
            m["timestamp"],             # then recency
            m["confidence"],            # then confidence
        ),
    )
```

## Detecting the conflict in the first place

Resolution only fires if you *notice* the contradiction. Two practical approaches:

- **Surface, then let the model judge.** Retrieve all candidates, hand them to the model *with their metadata* (timestamp, source), and instruct it: "These may conflict; prefer the most recent authoritative source and note the discrepancy." This is robust and cheap to build.
- **Resolve before injection.** Cluster retrieved facts by what they're *about* (same entity/attribute), and when a cluster disagrees, apply the policy above and inject only the winner. This keeps the prompt clean but needs logic to decide two facts are "about the same thing."

For long-term memory specifically, conflict resolution also happens on **write**: when you consolidate "user upgraded to pro," you don't just append it — you find the stale "free tier" memory and supersede or delete it, so recall never surfaces the contradiction. Append-only memory stores rot; consolidation that updates and invalidates is what keeps them coherent.

## Key takeaways

- Eager retrieval buries the signal and burns tokens; **just-in-time** retrieval — retrieval as a tool the agent calls when a step needs it — is the default for agent workloads.
- JIT costs an extra round-trip when retrieval is needed; eager pre-loading still wins when nearly every turn needs the same context.
- Real corpora and memory stores return **contradictions**; resolve them by **source authority, recency, and confidence**, which is why every fact needs `timestamp` and `source` metadata.
- Resolve conflicts on **recall** (surface metadata and let the model judge, or pick the winner) and on **write** (consolidate and supersede stale memories so they never resurface).
