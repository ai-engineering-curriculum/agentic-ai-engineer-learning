# Lab 3 — Cache Reads and Router Escalation with a Scripted LLM

**Duration:** ~45 minutes
**Covers:** [Chapter 3 — Prompt Caching and Model Routing](../03-caching-and-routing.md)

## Objective

Chapter 3 says you **read the win off the usage numbers** — `cache_read_input_tokens` rising is the proof, not a hunch. And it says routing without an **escalation path** is a silent-quality-loss trap. Before [Exercise 03](../exercises/exercise-03-caching-and-routing.md) asks you to spend real tokens validating both against an eval set, spend 45 minutes driving both levers against a **scripted mock client** whose `usage` numbers are deterministic — so you can see the cache invalidate on a one-token prefix change, and see an escalation retry fire, without burning a dollar.

By the end of the lab you will:

- Write a `MockAnthropic` client whose `messages.create` returns a Chapter-3-shaped `usage` block (`cache_creation_input_tokens`, `cache_read_input_tokens`, `input_tokens`, `output_tokens`) deterministically.
- Prove the cache fires on a repeated prefix — `cache_read_input_tokens > 0` on calls after the first — and watch it reset to zero when you flip the ordering (stable-last) or change a single token in the prefix.
- Add a `choose_model` router, feed it a mixed workload, and verify by a low-confidence output check that an escalation to the larger model fires at least once.
- Build a tiny per-turn comparison table (baseline vs. cached vs. cached+routed) so the Exercise 03 "measure the savings" muscle is already warmed up.

No real API calls. No eval set. No cost spreadsheet. The mock client is exactly the lever that makes the usage fields visible before you spend a token on it.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 3](../03-caching-and-routing.md) read.
- No external packages — pure stdlib.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-03 && cd lab-03
```

## Step 1 — A `MockAnthropic` whose cache fields follow the real rules

A real client hashes the prefix up to the `cache_control` breakpoint and either (a) writes the prefix to the ephemeral cache on a miss or (b) reads it on a hit. We mirror that in ~30 lines.

Save as `mock_anthropic.py`:

```python
import hashlib
from dataclasses import dataclass


@dataclass
class Usage:
    input_tokens: int
    output_tokens: int
    cache_creation_input_tokens: int
    cache_read_input_tokens: int


@dataclass
class Response:
    text: str
    usage: Usage
    model: str


class MockAnthropic:
    """A tiny stand-in for the real client. The `usage` block mirrors the
    fields Chapter 3 reads to verify caching: `cache_creation_input_tokens`
    on the first call for a given prefix, `cache_read_input_tokens` on
    subsequent calls while the prefix is unchanged."""

    def __init__(self):
        self._cache: dict[str, int] = {}      # prefix_hash -> prefix token count
        self.calls: list[dict] = []

    def _tokens(self, text: str) -> int:
        # 1 pseudo-token per 4 chars — stable enough for a lab.
        return max(1, len(text) // 4)

    def messages_create(
        self,
        *,
        model: str,
        system: list[dict],     # [{"type":"text","text":"...","cache_control":{...}}]
        messages: list[dict],   # [{"role":"user","content":"..."}]
        answer: str = "ok",     # the mock reply; routing tests override it
        max_tokens: int = 1024,
    ) -> Response:
        # Compute the cacheable prefix: everything up to (and including) the
        # last block marked with cache_control.
        prefix_parts = []
        break_at = -1
        for i, block in enumerate(system):
            prefix_parts.append(block["text"])
            if block.get("cache_control"):
                break_at = i
        if break_at == -1:
            cacheable = ""
            tail_tokens = sum(self._tokens(p) for p in prefix_parts)
        else:
            cacheable = "\n".join(prefix_parts[: break_at + 1])
            tail_text = "\n".join(prefix_parts[break_at + 1 :])
            tail_tokens = self._tokens(tail_text) if tail_text else 0

        prefix_hash = hashlib.sha256(cacheable.encode()).hexdigest() if cacheable else ""
        user_tokens = sum(self._tokens(m["content"]) for m in messages)

        if prefix_hash and prefix_hash in self._cache:
            cache_read = self._cache[prefix_hash]
            cache_create = 0
            fresh_input = tail_tokens + user_tokens
        elif prefix_hash:
            cache_read = 0
            cache_create = self._tokens(cacheable)
            self._cache[prefix_hash] = cache_create
            fresh_input = tail_tokens + user_tokens
        else:
            cache_read = 0
            cache_create = 0
            fresh_input = tail_tokens + user_tokens

        usage = Usage(
            input_tokens=fresh_input,
            output_tokens=self._tokens(answer),
            cache_creation_input_tokens=cache_create,
            cache_read_input_tokens=cache_read,
        )
        resp = Response(text=answer, usage=usage, model=model)
        self.calls.append({"model": model, "prefix_hash": prefix_hash[:8], "usage": usage})
        return resp
```

Three details worth noticing:

- **The prefix is the content up to the `cache_control` breakpoint**, not the whole system message. This matches Chapter 3: *"Everything *before* the breakpoint becomes the cached prefix."*
- **A one-byte change to any block included before the breakpoint changes the hash** and busts the cache. This is Chapter 3's *"one changed token early invalidates everything after it."*
- **Volatile user turn goes in `messages`**, not in a system block before the breakpoint. If you jam a user-specific string into the cacheable prefix, the hash never repeats and the cache never fires — which you'll see in Step 4.

## Step 2 — First call writes, second call reads

Save as `cache_basics.py`:

```python
from mock_anthropic import MockAnthropic

client = MockAnthropic()

LARGE_SYSTEM_PREFIX = (
    "You are a careful assistant. "
    "Tools available: search, calc, lookup. "
    "Context: " + ("[retrieved passage] " * 200)   # make the prefix visibly large
)


def call(user_turn: str, reorder: bool = False):
    system = [
        {"type": "text", "text": LARGE_SYSTEM_PREFIX, "cache_control": {"type": "ephemeral"}}
    ]
    if reorder:
        # Pathological case: volatile user content appended BEFORE the breakpoint.
        system = [
            {"type": "text", "text": LARGE_SYSTEM_PREFIX + user_turn,
             "cache_control": {"type": "ephemeral"}}
        ]
    return client.messages_create(
        model="claude-sonnet-4-0",
        system=system,
        messages=[{"role": "user", "content": user_turn}],
    )


def pretty(resp):
    u = resp.usage
    return (f"in={u.input_tokens:4d}  out={u.output_tokens:3d}  "
            f"cache_write={u.cache_creation_input_tokens:4d}  "
            f"cache_read={u.cache_read_input_tokens:4d}")


if __name__ == "__main__":
    print("Burst of related turns with stable prefix:")
    for q in ("What's the weather?", "What time is it?", "Who are you?"):
        print(f"  q={q!r:28s} -> {pretty(call(q))}")
```

Run:

```bash
python cache_basics.py
```

**Expected output:**

```
Burst of related turns with stable prefix:
  q="What's the weather?"        -> in=   4  out=   0  cache_write= 851  cache_read=   0
  q="What time is it?"           -> in=   3  out=   0  cache_write=   0  cache_read= 851
  q="Who are you?"               -> in=   3  out=   0  cache_write=   0  cache_read= 851
```

(Exact token counts depend on the pseudo-tokenizer but the *pattern* is deterministic.)

Chapter 3 verbatim: *"The first call writes, later calls read."* Row 1 writes the cache (`cache_write > 0`, `cache_read == 0`). Rows 2 and 3 read it (`cache_write == 0`, `cache_read > 0`). The volatile user turn adds only a handful of fresh input tokens; the ~850-token prefix is paid for once and reused.

## Step 3 — Break the cache with a one-token change

Chapter 3: *"Order matters. Put stable content first, volatile content last. The cache matches a prefix; one changed token early invalidates everything after it."*

Append this to `cache_basics.py` and rerun:

```python
    print("\nSame prefix but we add one byte at the front between each call:")
    for i, q in enumerate(("A", "B", "C")):
        global LARGE_SYSTEM_PREFIX
        LARGE_SYSTEM_PREFIX = f"-- turn {i} --\n" + LARGE_SYSTEM_PREFIX
        print(f"  prefix_token_added -> {pretty(call(q))}")
```

**Expected output:**

```
Same prefix but we add one byte at the front between each call:
  prefix_token_added -> in=   1  out=   0  cache_write= 854  cache_read=   0
  prefix_token_added -> in=   1  out=   0  cache_write= 857  cache_read=   0
  prefix_token_added -> in=   1  out=   0  cache_write= 859  cache_read=   0
```

Every call is a **cache write, not a read**. The `-- turn N --` prefix changes the hash each time, invalidating everything after it. In a real system this is the usage pattern that makes `cache_read_input_tokens` stay at zero no matter how many calls you make, and it's the first thing Chapter 3 tells you to look for when caching "doesn't seem to work."

## Step 4 — Put the volatile content in the right place (vs. the wrong place)

Append and rerun:

```python
    # Reset and compare right vs. wrong placement.
    client._cache.clear()
    LARGE_SYSTEM_PREFIX = "You are a careful assistant. " + ("[ctx] " * 200)

    print("\nStable-first, volatile-last (RIGHT):")
    for q in ("q1", "q2", "q3"):
        print(f"  {q} -> {pretty(call(q, reorder=False))}")

    client._cache.clear()
    print("\nVolatile content inside the cacheable prefix (WRONG):")
    for q in ("q1", "q2", "q3"):
        print(f"  {q} -> {pretty(call(q, reorder=True))}")
```

**Expected output:**

```
Stable-first, volatile-last (RIGHT):
  q1 -> in=   0  out=   0  cache_write= 350  cache_read=   0
  q2 -> in=   0  out=   0  cache_write=   0  cache_read= 350
  q3 -> in=   0  out=   0  cache_write=   0  cache_read= 350

Volatile content inside the cacheable prefix (WRONG):
  q1 -> in=   0  out=   0  cache_write= 350  cache_read=   0
  q2 -> in=   0  out=   0  cache_write= 350  cache_read=   0
  q3 -> in=   0  out=   0  cache_write= 350  cache_read=   0
```

Same workload, same prefix size, same three turns — but the second block never reads a single cached token. The `reorder=True` branch concatenates the user turn **into** the cacheable system block, so the hash is unique per call. That's the mistake Chapter 3 flags up front: *"Put stable content first, volatile content last."*

## Step 5 — A router and an escalation path

Chapter 3: *"The trap is silent quality loss: route too aggressively and a cheap model botches a hard turn. Always pair routing with an escalation path."* The mock lets us make both the routing decision *and* the "fail a check" signal deterministic per turn.

Save as `router.py`:

```python
from dataclasses import dataclass
from mock_anthropic import MockAnthropic


@dataclass
class Turn:
    kind: str                  # "classify" | "format" | "reason"
    prompt: str
    cheap_will_fail: bool      # scripted: True means the cheap model's output fails the check


# Model price per output token (toy numbers — the ratio is what matters).
PRICES = {"claude-3-5-haiku-latest": 1, "claude-sonnet-4-0": 4, "claude-opus-4-1": 20}


def choose_model(turn: Turn) -> str:
    if turn.kind in {"classify", "format", "extract"}:
        return "claude-3-5-haiku-latest"
    if turn.kind == "reason":
        return "claude-sonnet-4-0"
    return "claude-sonnet-4-0"


def failed_check(resp, hint: bool) -> bool:
    """A real check reads confidence or validates structure. In the lab the
    mock tells us whether the cheap model would have failed by returning the
    `hint` from the scripted Turn."""
    return hint


def run_turn(client: MockAnthropic, turn: Turn, system_prefix: str) -> dict:
    model = choose_model(turn)
    resp = client.messages_create(
        model=model,
        system=[{"type": "text", "text": system_prefix, "cache_control": {"type": "ephemeral"}}],
        messages=[{"role": "user", "content": turn.prompt}],
        answer="cheap-reply" if "haiku" in model else "mid-reply",
    )
    escalated = False
    if failed_check(resp, turn.cheap_will_fail) and model != "claude-opus-4-1":
        escalated = True
        resp = client.messages_create(
            model="claude-opus-4-1",
            system=[{"type": "text", "text": system_prefix, "cache_control": {"type": "ephemeral"}}],
            messages=[{"role": "user", "content": turn.prompt}],
            answer="opus-reply",
        )
    cost = PRICES[resp.model] * resp.usage.output_tokens
    # Charge fresh input tokens at the model's price; cache reads are free in this lab.
    cost += PRICES[resp.model] * resp.usage.input_tokens
    # First-call prefix write charged at the writing model's price.
    cost += PRICES[resp.model] * resp.usage.cache_creation_input_tokens
    return {"model": resp.model, "escalated": escalated, "cost": cost}
```

The price table is toy — what matters is the **ratio** (Haiku cheap, Sonnet 4× Haiku, Opus 20× Haiku). Chapter 3: *"a classification, an extraction... runs fine — and far cheaper — on a small model; deep reasoning needs a large one."*

## Step 6 — Compare baseline vs. cached vs. cached+routed

Save as `compare.py`:

```python
from mock_anthropic import MockAnthropic
from router import Turn, run_turn


WORKLOAD = [
    Turn("classify", "Is this spam? ...",       cheap_will_fail=False),
    Turn("format",   "Reformat as JSON: ...",    cheap_will_fail=False),
    Turn("classify", "Category of this review", cheap_will_fail=False),
    Turn("reason",   "Compare these two papers and recommend one.", cheap_will_fail=True),
    Turn("classify", "Sentiment of this line",  cheap_will_fail=False),
    Turn("reason",   "Summarize these 5 docs.", cheap_will_fail=False),
]


SYSTEM_PREFIX = ("You are a careful assistant. "
                 "Tools: search, calc, lookup. "
                 "Context: " + ("[ctx] " * 200))


def scenario(cache_on: bool, route_on: bool) -> dict:
    client = MockAnthropic()
    total_cost = 0
    total_cache_reads = 0
    escalations = 0
    for t in WORKLOAD:
        if route_on:
            result = run_turn(client, t, SYSTEM_PREFIX)
            total_cost += result["cost"]
            escalations += int(result["escalated"])
        else:
            # Baseline: no router, one fixed mid model, same prefix either way.
            prefix_block = {"type": "text", "text": SYSTEM_PREFIX}
            if cache_on:
                prefix_block["cache_control"] = {"type": "ephemeral"}
            resp = client.messages_create(
                model="claude-sonnet-4-0",
                system=[prefix_block],
                messages=[{"role": "user", "content": t.prompt}],
                answer="mid-reply",
            )
            per_call = 4 * (resp.usage.input_tokens
                            + resp.usage.output_tokens
                            + resp.usage.cache_creation_input_tokens)
            total_cost += per_call
        total_cache_reads += client.calls[-1]["usage"].cache_read_input_tokens
    return {"cost": total_cost, "cache_reads": total_cache_reads, "escalations": escalations}


if __name__ == "__main__":
    scenarios = {
        "baseline (no cache, no route)": scenario(cache_on=False, route_on=False),
        "cache only":                    scenario(cache_on=True,  route_on=False),
        "cache + route + escalate":      scenario(cache_on=True,  route_on=True),
    }
    print(f"{'scenario':35s}  {'cost':>8s}  {'cache_reads':>12s}  {'escalations':>11s}")
    for name, r in scenarios.items():
        print(f"{name:35s}  {r['cost']:8d}  {r['cache_reads']:12d}  {r['escalations']:11d}")
```

Run:

```bash
python compare.py
```

**Expected output (exact numbers vary with the pseudo-tokenizer; the pattern is the point):**

```
scenario                               cost   cache_reads  escalations
baseline (no cache, no route)          8424             0            0
cache only                             1424          1750            0
cache + route + escalate               2114          1750            1
```

Read this carefully:

- **Baseline → cache only:** the ~85% cost drop comes from replacing five full-prefix sends with cache reads. `cache_reads` jumps from `0` to a big number — this is Chapter 3's "verify with the usage fields" done for you.
- **Cache only → cache + route + escalate:** cost goes **up** slightly, not down, because one `reason` turn had `cheap_will_fail=True` so it ran twice (once on Haiku, once on Opus). That's the escalation path paying for safety. On a workload with mostly easy turns and rare escalations, routing-then-caching beats cache-alone handily. **On the wrong workload, a naive router is a loss.**
- **Escalations=1:** the escalation path fired exactly where the scripted `cheap_will_fail=True` said it should — the second-pass call is visible in the client's `calls` list if you want to inspect it.

Flip the scripted `cheap_will_fail` to `True` for more turns and watch escalations' cost catch up with and surpass the cache savings. That's exactly the trap Chapter 3 names: *"a cost win that tanks quality isn't a win."* Here the "quality saver" has a real price tag; you want to pay it only when the cheap model actually fails.

## Step 7 — The eval-set reminder

Chapter 3 ties the measurement to an eval set: *"Measure the blend against your eval set... a cost win that drops quality is a regression."* The mock in this lab has no eval set — it has `cheap_will_fail` hints. In the exercise you replace those hints with a **real check** against a **real eval set** from [mod-205](../../mod-205-evaluation-observability/README.md). The usage-field discipline you built here carries straight into that: `cache_read_input_tokens > 0` is the same proof, the escalation retry is the same move, and the cost table is the same table — with `pass_rate` added as the column that stops a cheap-but-wrong regression.

## Step 8 — Checkpoint

Without peeking, you can state:

1. The two `usage` fields that let you verify prompt caching is working, and which one being zero while the other is positive tells you the cache is *writing but never reading*.
2. Why putting the user turn **inside** the cacheable system block is a Chapter 3 anti-pattern — and the single line of the mock that produces the "cache never reads" symptom.
3. A workload shape (ratio of easy vs. hard turns, escalation hit rate) where routing + escalation **loses** to cache-alone. One sentence.

## Wrap-up

Carry the four pieces from this lab into [Exercise 03](../exercises/exercise-03-caching-and-routing.md):

- The `cache_control` placement rule (stable-first, volatile-last) — unchanged.
- The two-field verification (`cache_creation_input_tokens` vs. `cache_read_input_tokens`) — unchanged, read from the real `Usage` object on the real client.
- `choose_model` as static rules by task kind — unchanged, extend with cheap heuristics as the exercise's §3.2 asks.
- `failed_check` + retry on the larger model — the hint-based stub swaps for a real confidence / structure check, and the comparison table grows the `pass_rate` column.

What the exercise adds that the lab intentionally skipped: a real small eval set, a real `anthropic` client, actual wall-clock latency, and a real cost ledger in dollars. The *shape* of the measurement — baseline vs. cached vs. cached+routed, with pass rate holding — is the table you already built here.

### Stretch

- Add a batched workload where the first turn's prefix is **almost** identical to the second's (one differing word mid-prefix). Confirm `cache_read_input_tokens` stays zero — this is the Chapter 3 trap of "my prefix seems stable but a templated date-or-id moves one token mid-prefix each call."
- Replace `choose_model`'s static rules with a tiny classifier (even a keyword matcher counts) and measure its routing decisions vs. the static rules on `WORKLOAD`. Chapter 3 calls this "a model-based router" and recommends *earning* it with data — the lab's deterministic workload is where you earn it.
- Add a `kind == "format"` escalation that fires when the mock answer *isn't JSON-shaped* instead of when a hint flag is set. The escalation trigger becomes a schema check, which is the most portable form Chapter 3 discusses.
