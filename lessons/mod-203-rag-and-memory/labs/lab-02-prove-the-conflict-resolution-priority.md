# Lab 2 — Prove the Conflict-Resolution Priority Order

**Duration:** ~30 minutes
**Covers:** [Chapter 3 — Just-in-Time Retrieval and Conflict Resolution](../03-jit-retrieval-conflict.md)

## Objective

Chapter 3 gives a four-line conflict resolver:

```python
def resolve(conflicting: list[dict]) -> dict:
    return max(conflicting,
               key=lambda m: (SOURCE_RANK[m["source"]],
                              m["timestamp"],
                              m["confidence"]))
```

The order of the tuple is load-bearing: **source authority first, then recency, then confidence**. This lab proves it by constructing three contradictions where each signal is the decider, and one where the "obvious" answer (newer is better) is the wrong one.

You will:

- Run the chapter's `resolve` against contrived memory sets.
- Flip the tuple order and watch the wrong fact win.
- See the write-side consolidation rule in action (Chapter 3: "find the stale memory and supersede or delete it").

Pure Python, offline, no API. [Exercise 02](../exercises/exercise-02-agent-long-term-memory.md) is where you wire this resolver into a real agent's recall path.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- Chapter 3 read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-02 && cd lab-02
```

## Step 1 — Port the resolver

Save as `resolve.py`:

```python
SOURCE_RANK = {"user_stated": 2, "agent_inferred": 1}


def resolve(conflicting: list[dict]) -> dict:
    return max(
        conflicting,
        key=lambda m: (
            SOURCE_RANK[m["source"]],
            m["timestamp"],
            m["confidence"],
        ),
    )
```

ISO-8601 timestamp strings sort correctly as plain strings, so `max()` by `m["timestamp"]` is a legitimate "most recent wins." Note this detail — using arbitrary date formats would break the comparison silently.

## Step 2 — Case A: recency is the decider

Save as `case_a_recency.py`:

```python
from resolve import resolve

# Same source (user_stated), same confidence. Only the timestamps differ.
candidates = [
    {"fact": "user tier: free",
     "timestamp": "2026-10-10T09:00:00Z",
     "source": "user_stated", "confidence": 0.9},
    {"fact": "user tier: pro",
     "timestamp": "2026-11-15T14:30:00Z",
     "source": "user_stated", "confidence": 0.9},
]

winner = resolve(candidates)
print("winner:", winner["fact"])
```

Run:

```bash
python case_a_recency.py
```

**Expected output:**

```
winner: user tier: pro
```

**Why this matters:** when the first key (source rank) ties and so does the last (confidence), the middle key — recency — breaks the tie. Chapter 3's "the user upgraded" example reduces to exactly this case.

## Step 3 — Case B: source authority overrides recency

Save as `case_b_authority.py`:

```python
from resolve import resolve

# The agent's recent *inference* contradicts the user's older *direct statement*.
candidates = [
    {"fact": "user tier: pro",
     "timestamp": "2026-10-10T09:00:00Z",
     "source": "user_stated",    "confidence": 0.95},
    {"fact": "user tier: free",
     "timestamp": "2026-11-15T14:30:00Z",   # NEWER
     "source": "agent_inferred", "confidence": 0.60},
]

winner = resolve(candidates)
print("winner:", winner["fact"])
```

Run:

```bash
python case_b_authority.py
```

**Expected output:**

```
winner: user tier: pro
```

**Why this matters:** the newer memory would win on recency alone — but it comes from `agent_inferred` (rank 1), while the older one comes from `user_stated` (rank 2). The tuple is examined left-to-right, so the recency term is never consulted. This is Chapter 3's *"Recency is the default tiebreaker, but not absolute: a newer low-quality source shouldn't beat an older authoritative one"* written as code.

**Reorder experiment.** In `resolve.py`, temporarily swap the key tuple to `(m["timestamp"], SOURCE_RANK[m["source"]], m["confidence"])` — recency first. Rerun Case B:

```
winner: user tier: free
```

The *wrong* fact now wins. Put the original order back. This is the point of the lab: the three priorities aren't interchangeable — the chapter's order encodes a specific policy, and permuting it changes the agent's behavior.

## Step 4 — Case C: confidence breaks a true tie

Save as `case_c_confidence.py`:

```python
from resolve import resolve

# Same source, same timestamp — only confidence differs. Imagine two
# simultaneous agent inferences from different reasoning paths.
candidates = [
    {"fact": "user timezone: PST",
     "timestamp": "2026-11-15T14:30:00Z",
     "source": "agent_inferred", "confidence": 0.3},
    {"fact": "user timezone: EST",
     "timestamp": "2026-11-15T14:30:00Z",
     "source": "agent_inferred", "confidence": 0.8},
]

print("winner:", resolve(candidates)["fact"])
```

Run:

```bash
python case_c_confidence.py
```

**Expected output:**

```
winner: user timezone: EST
```

**Why this matters:** confidence is the third-tier tiebreaker — only consulted when both source rank **and** timestamp tie. Rare in practice, but it's the backstop that keeps the resolver deterministic.

## Step 5 — Case D: the signal the resolver can't fix

Construct a case where all three keys tie and the two facts still disagree. Save as `case_d_true_tie.py`:

```python
from resolve import resolve

candidates = [
    {"fact": "user tier: pro",
     "timestamp": "2026-11-15T14:30:00Z",
     "source": "user_stated", "confidence": 0.9},
    {"fact": "user tier: enterprise",
     "timestamp": "2026-11-15T14:30:00Z",
     "source": "user_stated", "confidence": 0.9},
]

print("winner:", resolve(candidates)["fact"])
print("(tied inputs — Python `max` returned the first one in insertion order)")
```

Run and observe which fact `max()` picks. Reorder the list and rerun. The winner flips because `max()` is stable — on ties it returns the first match.

**Why this matters:** Chapter 3's three signals aren't *infinite* — when all three tie, you have no principled winner and the resolver is arbitrary. Production policies add a fourth signal (e.g., a user re-confirmation prompt, or a flag that forces the agent to ask instead of pick) when stakes are high. Know where the policy runs out.

## Step 6 — The write-side rule: supersede, don't append

Chapter 3: *"Append-only memory stores rot; consolidation that updates and invalidates is what keeps them coherent."* Model the two policies side by side.

Save as `write_policies.py`:

```python
from resolve import resolve


def append_only_store():
    """Every write adds a row. Reads return all rows for the attribute."""
    return []


def consolidating_store():
    """On write, find the existing row for the same attribute and delete it
    before appending. Reads return at most one row per attribute."""
    return []


def write_append(store, mem):
    store.append(mem)


def write_consolidate(store, mem):
    attr = mem["fact"].split(":", 1)[0]
    store[:] = [m for m in store if not m["fact"].startswith(f"{attr}:")]
    store.append(mem)


facts = [
    {"fact": "user tier: free",
     "timestamp": "2026-10-10T09:00:00Z",
     "source": "user_stated", "confidence": 0.9},
    {"fact": "user tier: pro",
     "timestamp": "2026-11-15T14:30:00Z",
     "source": "user_stated", "confidence": 0.95},
]

a = append_only_store()
c = consolidating_store()
for f in facts:
    write_append(a, f)
    write_consolidate(c, f)

print("append-only rows:")
for m in a:
    print(" ", m["fact"], m["timestamp"])
print("  recall picks ->", resolve(a)["fact"])

print()
print("consolidating rows:")
for m in c:
    print(" ", m["fact"], m["timestamp"])
print("  recall picks ->", resolve(c)["fact"] if c else "(empty)")
```

Run:

```bash
python write_policies.py
```

**Expected output:**

```
append-only rows:
  user tier: free  2026-10-10T09:00:00Z
  user tier: pro   2026-11-15T14:30:00Z
  recall picks -> user tier: pro

consolidating rows:
  user tier: pro   2026-11-15T14:30:00Z
  recall picks -> user tier: pro
```

**Why this matters:** both policies return the *right answer* here because `resolve` cleans up the ambiguity at recall. But the append-only store still carries the stale `"free"` row — the next top-k search can surface it right next to the winner, and every recall has to re-decide the same conflict. Consolidation deletes it at write time, so the store stays coherent **without** the resolver having to do work.

## Step 7 — Adversarial checkpoint

Which of the following does **not** break `resolve`?

1. A memory with `source="crawler_guess"` (not in `SOURCE_RANK`).
2. A memory with `timestamp` formatted as `"11/15/2026"` instead of ISO-8601.
3. A memory with `confidence=None`.
4. Two memories with identical `(source, timestamp, confidence)` but different `fact` values.

Try each in a REPL. You should find:

- (1) raises `KeyError` — the chapter's `SOURCE_RANK` is explicit; unknown sources need a default.
- (2) sorts wrong silently — `"11/15/2026"` < `"2026-01-01"` as strings. Use ISO-8601.
- (3) raises `TypeError` on `max`.
- (4) is the Case D scenario — no error, but the winner is arbitrary.

**Checkpoint:** you can say *exactly* why the tuple order is `(source, timestamp, confidence)` and not any of its permutations — and which real-world failure each ordering would produce.

## Wrap-up

You've exercised every row of Chapter 3's priority rule against a contrived case and seen one case the resolver can't fix. Carry the `resolve` function into [exercise-02](../exercises/exercise-02-agent-long-term-memory.md), where it runs on top of a real vector store's recall path and alongside the consolidation step you just prototyped in Step 6.
