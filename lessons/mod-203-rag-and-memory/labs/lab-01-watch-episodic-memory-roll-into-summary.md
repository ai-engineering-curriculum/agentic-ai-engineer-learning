# Lab 1 — Watch Episodic Memory Roll Into a Summary

**Duration:** ~30 minutes
**Covers:** [Chapter 2 — Memory Systems for Agents](../02-agent-memory.md)

## Objective

Chapter 2 presents an `EpisodicMemory` class that keeps the most recent turns verbatim and rolls older ones into a running summary. You will:

- Instantiate the class, feed it a stream of turns, and watch `self.turns` and `self.summary` change over time.
- See exactly *when* the first summarization fires and *what* ends up in the summary versus what stays verbatim.
- Confirm the shape of the message list the agent would receive from `context()` on turn 1, turn 10, and turn 50.

This lab is pure Python and offline — no API keys, no vector DB. [Exercise 02](../exercises/exercise-02-agent-long-term-memory.md) is where you then wire a real LLM summarizer and a long-term tier behind it.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- Chapter 2 read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-01 && cd lab-01
```

No external packages required.

## Step 1 — Port the Chapter 2 class with a stub summarizer

Save as `episodic.py`:

```python
class EpisodicMemory:
    """From Chapter 2: keep the last `keep_recent` turns verbatim; roll the
    rest into a running summary string."""

    def __init__(self, keep_recent: int = 10, summarize=None):
        self.turns: list[dict] = []
        self.summary: str = ""
        self.keep_recent = keep_recent
        self._summarize = summarize or _stub_summarize

    def add(self, role: str, content: str) -> None:
        self.turns.append({"role": role, "content": content})
        if len(self.turns) > self.keep_recent:
            old = self.turns[: -self.keep_recent]
            self.summary = self._summarize(self.summary, old)
            self.turns = self.turns[-self.keep_recent :]

    def context(self) -> list[dict]:
        head = [{"role": "system", "content": f"Earlier: {self.summary}"}]
        return head + self.turns if self.summary else self.turns


def _stub_summarize(prev_summary: str, old_turns: list[dict]) -> str:
    """Deterministic placeholder for the LLM summarizer. Concatenates the
    role + first 40 chars of each old turn onto the prior summary so you can
    read it and see exactly what was absorbed."""
    bits = [f"{t['role']}:{t['content'][:40].strip()}" for t in old_turns]
    joined = " | ".join(bits)
    if prev_summary:
        return f"{prev_summary} || {joined}"
    return joined
```

The stub summarizer is deliberately not an LLM — in this lab you want to *see* exactly what moved from `self.turns` into `self.summary`, not read a paraphrase.

## Step 2 — Feed it three turns and confirm nothing summarizes yet

Save as `below_threshold.py`:

```python
from episodic import EpisodicMemory

mem = EpisodicMemory(keep_recent=10)
for i in range(1, 4):
    mem.add("user", f"user turn {i}: tell me about X{i}")
    mem.add("assistant", f"assistant turn {i}: here is X{i}")

print("turns kept:", len(mem.turns))
print("summary:", repr(mem.summary))
print("context() length:", len(mem.context()))
```

Run:

```bash
python below_threshold.py
```

**Expected output:**

```
turns kept: 6
summary: ''
context() length: 6
```

**Why this matters:** six messages (three user + three assistant) sit under the `keep_recent=10` threshold, so `self.summary` is still empty and `context()` returns exactly the turn list with **no** leading "Earlier:" system message. This is the baseline — nothing surprising yet.

## Step 3 — Cross the threshold and watch the first roll-up

Save as `first_rollup.py`:

```python
from episodic import EpisodicMemory

mem = EpisodicMemory(keep_recent=10)

for i in range(1, 7):                    # 12 messages, two per turn → cross 10
    mem.add("user",      f"u{i}: please do task {i}")
    mem.add("assistant", f"a{i}: did task {i}")

print(f"after {2*6} messages:")
print("  turns kept:   ", len(mem.turns))
print("  summary head: ", mem.summary[:120])
print("  context[0]:   ", mem.context()[0])
```

Run:

```bash
python first_rollup.py
```

**Expected output (shape — the summary content is deterministic given the stub):**

```
after 12 messages:
  turns kept:    10
  summary head:  user:u1: please do task 1 | assistant:a1: did task 1
  context[0]:    {'role': 'system', 'content': 'Earlier: user:u1: please do task 1 | assistant:a1: did task 1'}
```

**Why this matters:** on the 11th `add()` call, `len(self.turns)` becomes 11, exceeding `keep_recent=10`, so the oldest **1** message is pulled out, run through `_summarize`, and the summary is updated. On the 12th call, the next oldest is also pulled out. `self.turns` now holds *only the 10 most recent messages* and `context()` prepends an `Earlier:` system note — the exact two-part structure Chapter 2 shows.

## Step 4 — Let it run 50 turns and read the shape

Save as `long_horizon.py`:

```python
from episodic import EpisodicMemory

mem = EpisodicMemory(keep_recent=10)
for i in range(1, 26):                  # 50 messages total
    mem.add("user",      f"u{i}: ask {i}")
    mem.add("assistant", f"a{i}: answer {i}")

ctx = mem.context()
print("total messages ingested: 50")
print("turns kept verbatim:    ", len(mem.turns))
print("summary length (chars): ", len(mem.summary))
print("context() length:       ", len(ctx))
print()
print("--- verbatim tail (most recent two turns) ---")
for m in mem.turns[-4:]:
    print(" ", m)
print()
print("--- earliest content now only lives in the summary ---")
print("  'u1' in summary?   ", "u1:" in mem.summary)
print("  'u1' in turns?     ", any("u1:" in m["content"] for m in mem.turns))
print("  'u25' in turns?    ", any("u25:" in m["content"] for m in mem.turns))
print("  'u25' in summary?  ", "u25:" in mem.summary)
```

Run:

```bash
python long_horizon.py
```

**Expected output:**

```
total messages ingested: 50
turns kept verbatim:     10
summary length (chars):  ~1700   (your exact number may differ by a few chars)
context() length:        11

--- verbatim tail (most recent two turns) ---
  {'role': 'user',      'content': 'u24: ask 24'}
  {'role': 'assistant', 'content': 'a24: answer 24'}
  {'role': 'user',      'content': 'u25: ask 25'}
  {'role': 'assistant', 'content': 'a25: answer 25'}

--- earliest content now only lives in the summary ---
  'u1' in summary?    True
  'u1' in turns?      False
  'u25' in turns?     True
  'u25' in summary?   False
```

**Why this matters:** after 50 messages the class holds exactly the invariant Chapter 2 promises — the tail of 10 messages verbatim plus one summary string. `u1` is in the summary only; `u25` is in the turns only. The agent sees 11 messages instead of 50, and that is how the context window stops being the bottleneck.

## Step 5 — Compare `context()` against a naive "replay everything" baseline

Add to `long_horizon.py`:

```python
naive_ctx = [{"role": m["role"], "content": m["content"]}
             for m in [*[{"role": "user", "content": f"u{i}: ask {i}"}
                         for i in range(1, 26)],
                       *[{"role": "assistant", "content": f"a{i}: answer {i}"}
                         for i in range(1, 26)]]]
print()
print("naive full-replay message count: ", len(naive_ctx))
print("episodic context() message count:", len(ctx))
print("reduction:", f"{(1 - len(ctx)/len(naive_ctx)) * 100:.0f}%")
```

Rerun. **Expected output:**

```
naive full-replay message count:  50
episodic context() message count: 11
reduction: 78%
```

**Why this matters:** this is Chapter 2's "replaying the full transcript doesn't scale — a 200-turn conversation won't fit" written as a number. On a 50-turn conversation with `keep_recent=10`, episodic memory cuts the message count by about four-fifths, and the ratio only gets better as the conversation grows.

## Step 6 — Adversarial checkpoint: what `keep_recent` is too small?

In a Python REPL, run:

```python
from episodic import EpisodicMemory
mem = EpisodicMemory(keep_recent=1)
mem.add("user", "my name is Alex and I prefer metric units")
mem.add("assistant", "got it — metric units it is")
mem.add("user", "what's the weather?")
print(mem.context())
```

Read the output. The assistant's acknowledgement has already been rolled into the summary string, and `mem.turns` holds just the latest user question. On the next turn the model sees a *paraphrased* "I confirmed metric" rather than the exact "got it — metric units it is." For a stub summarizer that's fine — for a real LLM summarizer it's lossy.

**Checkpoint:** you can state why `keep_recent` is a lever for **fidelity vs. token cost**: too small and the model sees a paraphrase of what *just happened* (fidelity loss); too large and every turn pays for a transcript that's partly redundant with the summary. Chapter 2's "10" is a starting point, not a fixed answer.

## Wrap-up

You have seen episodic memory's two invariants directly:

- Below `keep_recent` messages — no summary, `context()` is the raw list.
- At or above `keep_recent` — the overflow is summarized, `context()` is `[system(Earlier: …), …recent turns]`, and the message count stops growing linearly.

Carry the `EpisodicMemory` class forward into [exercise-02](../exercises/exercise-02-agent-long-term-memory.md), replace `_stub_summarize` with a real LLM call, and layer a long-term tier underneath so durable facts survive even when the summary itself falls out of a later rollup.
