# Lab 2 — Trace a ReAct Step with a Mock LLM

**Duration:** ~30 minutes
**Covers:** [Chapter 3 — The ReAct Loop: Thought, Action, Observation](../03-react-loop.md)

## Objective

Before Exercise 01 asks you to build a full ReAct agent against a real model, spend half an hour hand-stepping through one iteration of the loop with a **mocked LLM** that returns a hard-coded chunk. You will:

- Parse `Thought:` / `Action:` / `Action Input:` out of a raw model chunk.
- Dispatch the parsed action to a Python function and render the observation line that would be appended to the next prompt.
- Watch what changes when the mock emits `Final Answer:` vs. another `Action:`.
- See the three things Chapter 3 says you will get wrong the first time — stop sequence, parsing, step budget — up close, without burning any tokens.

This lab is pure Python, no API calls. It is intentionally narrower than Exercise 01 (no tests, no logging, no token counter, no real model) so that nothing hides the parsing primitives.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- Chapter 3 read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-02 && cd lab-02
```

No external packages needed.

## Step 1 — A mock LLM you can trust

Save as `mock_llm.py`:

```python
class MockLLM:
    """Returns pre-canned completions in order. Respects `stop` by truncating
    the canned chunk at the first stop sequence it contains."""

    def __init__(self, chunks):
        self._chunks = list(chunks)
        self.calls = []

    def complete(self, prompt, stop=None):
        chunk = self._chunks.pop(0)
        self.calls.append({"prompt": prompt, "stop": stop})
        if stop:
            for s in stop:
                i = chunk.find(s)
                if i != -1:
                    chunk = chunk[:i]
                    break
        return chunk
```

**Why a mock:** real providers may ignore your `stop` sequence under some sampling regimes. A deterministic mock lets you *prove* the parser works before you spend a token on a flaky trajectory.

## Step 2 — The parsers from Chapter 3

Save as `parse.py`:

```python
import json
import re

THOUGHT_RE = re.compile(r"Thought:\s*(.+)")
ACTION_RE  = re.compile(r"Action:\s*(\w+)")
INPUT_RE   = re.compile(r"Action Input:\s*(\{.*?\})", re.DOTALL)
FINAL_RE   = re.compile(r"Final Answer:\s*(.+)", re.DOTALL)


def parse_step(chunk):
    """Return either ('final', text) or ('action', name, args_dict).
    Raise ValueError on malformed input — the caller surfaces it as an observation."""
    if m := FINAL_RE.search(chunk):
        return ("final", m.group(1).strip())
    a = ACTION_RE.search(chunk)
    i = INPUT_RE.search(chunk)
    if not a or not i:
        raise ValueError(f"malformed step:\n{chunk!r}")
    return ("action", a.group(1), json.loads(i.group(1)))
```

## Step 3 — One tool, dispatched by name

Save as `tools.py`:

```python
import ast
import operator as op

_BINOPS = {ast.Add: op.add, ast.Sub: op.sub, ast.Mult: op.mul,
           ast.Div: op.truediv, ast.Pow: op.pow, ast.USub: op.neg}


def _safe_eval(node):
    if isinstance(node, ast.Expression):
        return _safe_eval(node.body)
    if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
        return node.value
    if isinstance(node, ast.BinOp) and type(node.op) in _BINOPS:
        return _BINOPS[type(node.op)](_safe_eval(node.left), _safe_eval(node.right))
    if isinstance(node, ast.UnaryOp) and type(node.op) in _BINOPS:
        return _BINOPS[type(node.op)](_safe_eval(node.operand))
    raise ValueError(f"disallowed expression node: {ast.dump(node)}")


def calc(expression: str) -> str:
    """Evaluate a restricted arithmetic expression. No eval(), no builtins."""
    return str(_safe_eval(ast.parse(expression, mode="eval")))


TOOLS = {"calc": calc}
```

Note: Chapter 3 and Exercise 01 both explicitly forbid `eval()`. This matches that constraint.

## Step 4 — One step, start to finish

Save as `step.py`:

```python
from mock_llm import MockLLM
from parse import parse_step
from tools import TOOLS

mock = MockLLM([
    # Turn 1: the model picks a tool.
    "Thought: Divide India's pop by China's.\n"
    "Action: calc\n"
    'Action Input: {"expression": "1.45e9 / 1.41e9"}\n'
    "Observation: SHOULD_BE_TRUNCATED_BY_STOP",
    # Turn 2: after the runtime injects the real observation, the model wraps up.
    "Thought: I have enough to answer.\n"
    "Final Answer: About 1.03 — India is roughly 1.03× China.\n",
])

prompt = "Question: pop(India) / pop(China) ?\n"

for step in range(1, 5):
    chunk = mock.complete(prompt, stop=["Observation:"])
    prompt += chunk
    kind, *rest = parse_step(chunk)
    if kind == "final":
        print(f"[step {step}] FINAL: {rest[0]}")
        break
    name, args = rest
    tool = TOOLS.get(name)
    if tool is None:
        obs = f"Error: unknown tool '{name}'."
    else:
        try:
            obs = tool(**args)
        except Exception as e:
            obs = f"Error: {e!r}"
    print(f"[step {step}] action={name} args={args} obs={obs!r}")
    prompt += f"\nObservation: {obs}\n"
else:
    raise RuntimeError("max_steps exhausted")
```

Run:

```bash
python step.py
```

**Expected output:**

```
[step 1] action=calc args={'expression': '1.45e9 / 1.41e9'} obs='1.0283687943262412'
[step 2] FINAL: About 1.03 — India is roughly 1.03× China.
```

**Why this matters:** the loop is small — mock → parse → dispatch → observe → repeat. The `SHOULD_BE_TRUNCATED_BY_STOP` string never appears in the output because `MockLLM.complete` honored the `stop=["Observation:"]` sequence. That is the Chapter 3 trick: **the runtime, not the model, writes the observation line.**

## Step 5 — The three things Chapter 3 says you will get wrong

Try each of these one at a time and note the behavior:

**(a) Forgetting the stop sequence.** Replace the first mock chunk with a version whose fake observation says something misleading, and call `mock.complete(prompt, stop=None)`. The loop will now parse the model's **invented** observation instead of running `calc`. That is the "model hallucinates its own observations" failure Chapter 3 warns about.

**(b) Bad parse.** Replace `Action Input: {"expression": "1.45e9 / 1.41e9"}` with `Action Input: expression = 1.45e9 / 1.41e9` (not JSON). The parser raises `ValueError`. In a production loop you would catch that exception and feed its message back as the next `Observation:` — "errors are observations" (Chapter 3). Wrap the `parse_step` call in `try/except ValueError` and confirm the next turn sees a diagnostic observation.

**(c) No step budget.** Give the mock a chunk list that never returns `Final Answer:` (e.g., three identical `calc` steps). Remove the `for step in range(1, 5):` cap and run under a `timeout 10s python step.py` wrapper. Note how fast you could rack up tokens against a real provider. Put the budget back.

## Step 6 — Adversarial checkpoint

Feed the mock this chunk as the first turn:

```
"Thought: let me search.\n"
"Action: search\n"
'Action Input: {"query": "not json\n'   # note: missing closing brace and quote
```

Confirm your loop raises `ValueError` **and** that you can wrap the raise so the agent gets an `Observation: Error: malformed step:...`. This is the minimum "errors are observations" behavior Exercise 01 later tests for.

**Checkpoint:** you can draw the Chapter 3 loop on a napkin — model call with `stop=["Observation:"]`, parse, dispatch, append `Observation:`, repeat until `Final Answer:` or budget — and you've seen each piece break in isolation.

## Wrap-up

Carry the three files from this lab into Exercise 01. Replace `MockLLM` with a real provider client, bolt on logging / tests / the token counter from Lab 1, and you have the Exercise 01 skeleton. The parsing, dispatch, and stop-sequence pieces are unchanged — because they are the whole agent.
