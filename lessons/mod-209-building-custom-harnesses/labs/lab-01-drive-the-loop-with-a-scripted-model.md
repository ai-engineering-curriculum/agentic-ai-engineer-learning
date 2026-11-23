# Lab 1 — Drive the Loop with a Scripted Model

**Duration:** ~45 minutes
**Covers:** [Chapter 1 — Harness Architecture](../01-harness-architecture.md), [Chapter 2 — The Agent Loop](../02-the-agent-loop.md)

## Objective

Chapter 2 claims "a harness is mostly code around the model: validation, permissions, hooks, execution, and stop conditions." Before [Exercise 01](../exercises/exercise-01-build-a-minimal-harness.md) asks you to build a full harness with real tools, a permission policy, hooks, and stop conditions, spend 45 minutes driving the **smallest possible** loop against a scripted model so the failure-as-observation shape is visible — no network, no API key, no live agent.

By the end of the lab you will:

- Build a ~60-line loop with the four immutable types from Chapter 1 and a two-tool registry of pure functions.
- Play five scripted turns through it: a happy read, an unknown tool, a schema-invalid input, a tool exception, and a path escape. Watch each become a `ToolResult` the loop appends and sends back, instead of a Python crash.
- Feel Chapter 2's rule "every tool call gets a result" by removing one result in the middle of history and seeing the provider-shape invariant break.
- Add the "repeated identical tool calls" stop condition from Chapter 2 and watch a stuck loop exit on purpose instead of running forever.

Pure stdlib; no SDK; no pytest; no `.claude/settings.json` wiring. Those additions are what Exercise 01 builds on top.

## Prerequisites

- Python 3.11+.
- [Chapter 1](../01-harness-architecture.md) and [Chapter 2](../02-the-agent-loop.md) read.
- No external packages — the whole lab uses `dataclasses`, `pathlib`, and `jsonschema` is simulated with a 10-line pure-Python validator.

## Setup

```bash
mkdir lab-01 && cd lab-01
mkdir workspace
echo "hello from app.py" > workspace/app.py
```

No venv needed.

## Step 1 — The four immutable types

Chapter 1 says: "A small set of immutable types carries through the whole harness. Getting them right early saves rework." Save as `types.py`:

```python
"""The core types from Chapter 1 §Data model."""
from dataclasses import dataclass
from typing import Any, Literal, Mapping


@dataclass(frozen=True)
class ToolCall:
    id: str
    name: str
    input: Mapping[str, Any]


@dataclass(frozen=True)
class ToolResult:
    call_id: str
    content: str
    is_error: bool = False


@dataclass(frozen=True)
class ModelTurn:
    text: str
    tool_calls: tuple[ToolCall, ...]
    stop_reason: Literal["end_turn", "tool_use", "max_tokens"]


@dataclass(frozen=True)
class RunResult:
    final_text: str
    stop_reason: Literal["done", "max_turns", "unknown_tool", "loop_detected"]
    turns: int
    transcript: tuple[Any, ...]
```

Two details worth noticing:

- **`frozen=True` on every dataclass.** Chapter 1: "Keep a small set of immutable types." Mutating `history` in place is one of the easiest ways to make a transcript unreplayable; immutable types are a small push against that.
- **`ToolResult.is_error` is a boolean, not a string.** Chapter 2: *"Errors are observations."* The content carries the human-readable reason; the flag is what lets the next model turn branch on "did my last call fail?" without string-matching.

## Step 2 — Two pure-function tools in a registry

A registry maps names to `(fn, schema)`. Keep the tools pure so the lab's focus stays on dispatch, not on files. Save as `registry.py`:

```python
"""A two-tool registry: read_file (pure function over an in-memory FS) and echo."""
from pathlib import Path

WORKSPACE = Path(__file__).parent / "workspace"


def read_file(path: str) -> str:
    """Read a file inside the workspace. Rejects paths that escape it."""
    requested = (WORKSPACE / path).resolve()
    if not requested.is_relative_to(WORKSPACE.resolve()):
        raise PermissionError(f"{path!r} is outside the workspace")
    return requested.read_text()


def echo(message: str) -> str:
    return message


SCHEMAS = {
    "read_file": {"required": ["path"], "properties": {"path": {"type": "string"}}},
    "echo":      {"required": ["message"], "properties": {"message": {"type": "string"}}},
}


REGISTRY = {"read_file": read_file, "echo": echo}
```

Three details worth noticing:

- **`resolve()` first, then check containment.** Chapter 2: *"`path.startswith(root)` fails on `../`, on prefixes, and on symlinks. Resolve first, then compare."* Try `read_file("../../etc/passwd")` later and see this fire.
- **`SCHEMAS` is intentionally tiny.** We're going to validate "required keys present, no extras" in ten lines of Python rather than pull in `jsonschema`. The point of the lab is the dispatch loop, not JSON Schema completeness.
- **The tool raises `PermissionError`** — the loop will catch any exception and turn it into a `ToolResult(is_error=True)`. Tools don't need to know about tool results.

## Step 3 — The minimal validator

A ten-line schema check lets us observe one more failure path without a dependency. Add to `registry.py`:

```python
def validate(name: str, args: dict) -> str | None:
    """Return None if args match schema, else a human-readable problem."""
    schema = SCHEMAS.get(name)
    if schema is None:
        return f"unknown tool {name!r}; available: {sorted(SCHEMAS)}"
    missing = [k for k in schema["required"] if k not in args]
    if missing:
        return f"missing required args: {missing}"
    extras = [k for k in args if k not in schema["properties"]]
    if extras:
        return f"unknown args: {extras}; allowed: {sorted(schema['properties'])}"
    return None
```

Two details:

- **Unknown tool is a schema problem, not a crash.** Chapter 2 is explicit that the error message should list the real tool names so the model's next turn has somewhere to go.
- **Extras are rejected.** A model that passes `{"path": "app.py", "pth": "app.py"}` because it's uncertain about the spelling should see `unknown args: ['pth']`, not have the typo quietly ignored and then wonder why nothing changed.

## Step 4 — The scripted model

Chapter 1's "pluggable edges" rule: the loop shouldn't know which model it's using. A scripted model that returns a fixed sequence of turns is all you need to test the loop end to end. Save as `scripted_model.py`:

```python
"""Replays a scripted list of ModelTurns; returns one per call to complete()."""
from types import SimpleNamespace
from typing import Iterator
from types_mod import ModelTurn  # see next step for the import


class ScriptedModel:
    def __init__(self, script: list[ModelTurn]) -> None:
        self._iter: Iterator[ModelTurn] = iter(script)

    def complete(self, system: str, messages: list, tools: list) -> ModelTurn:
        try:
            return next(self._iter)
        except StopIteration:
            raise AssertionError("scripted model ran out of turns; add another to the script")
```

Rename your `types.py` to `types_mod.py` so the import doesn't shadow stdlib `types`:

```bash
mv types.py types_mod.py
```

Three details:

- **`complete()` ignores `system`, `messages`, and `tools`** on purpose. The scripted model is deterministic; the whole point is that the *loop's* behavior is now a function only of the script, so one test can prove one property.
- **Running out of script is an `AssertionError`.** Chapter 1: *"deterministic tests for a non-deterministic system."* A loop that sends one more turn than the script expects is a bug in the loop or the test — not a silent "model returned nothing."
- **Why not a generator function?** You could, but a class with an iterator gives you a hook to add state later (call count, sleep, deliberate retryable errors) without rewriting the test script.

## Step 5 — The loop

Save as `loop.py`. This is the only non-trivial file in the lab:

```python
"""A ~60-line agent loop. Errors are observations; every call gets a result."""
from types_mod import RunResult, ToolCall, ToolResult
from registry import REGISTRY, validate


def execute(call: ToolCall) -> ToolResult:
    """Every failure path returns a ToolResult; the loop never crashes on a tool call."""
    problem = validate(call.name, dict(call.input))
    if problem:
        return ToolResult(call.id, f"invalid tool call: {problem}", is_error=True)
    tool = REGISTRY[call.name]
    try:
        output = tool(**call.input)
    except Exception as exc:
        return ToolResult(call.id, f"{type(exc).__name__}: {exc}", is_error=True)
    return ToolResult(call.id, str(output))


def _recent_calls(history: list, n: int) -> list[tuple[str, tuple]]:
    """The last n tool calls as (name, sorted-items-of-input) tuples, for loop detection."""
    calls = [m["call"] for m in history if m.get("kind") == "tool_call"]
    return [(c.name, tuple(sorted(c.input.items()))) for c in calls[-n:]]


def run(model, task: str, max_turns: int = 10) -> RunResult:
    history: list = [{"kind": "user", "text": task}]
    for turn in range(1, max_turns + 1):
        reply = model.complete(system="", messages=history, tools=[])

        if not reply.tool_calls:
            history.append({"kind": "assistant", "text": reply.text})
            return RunResult(reply.text, "done", turn, tuple(history))

        # Dispatch every tool call; each gets a result, whether it succeeds, errors, or is invalid.
        results = []
        for call in reply.tool_calls:
            history.append({"kind": "tool_call", "call": call})
            result = execute(call)
            history.append({"kind": "tool_result", "result": result})
            results.append(result)

        # Loop detection: same (name, inputs) three times in a row.
        recent = _recent_calls(history, 3)
        if len(recent) == 3 and len(set(recent)) == 1:
            return RunResult("", "loop_detected", turn, tuple(history))

    return RunResult("", "max_turns", max_turns, tuple(history))
```

Four things to notice, from Chapter 2:

- **No tool calls means done.** Line 20: `if not reply.tool_calls`. Chapter 2: *"Don't look for magic words in the text."*
- **Every tool call gets a result.** The `for call in reply.tool_calls` loop appends exactly one `tool_result` per call, even when `execute` returned an error. Chapter 2: *"Provider APIs require each tool call to be paired with a result in the next message."*
- **History is append-only.** No in-place mutation; snapshotting history to a replay cassette would be a one-liner.
- **Loop detection sits outside the dispatch.** The chapter's four stop conditions (no tool calls, `max_turns`, token budget, repeated identical calls) all check the history *between* turns, not inside the tool call. Keeping stop conditions here means you can test each one with a hand-crafted script.

## Step 6 — Drive five scripted scenarios

Save as `drive.py`:

```python
"""Drive the loop through five scripted scenarios from Chapter 2."""
from scripted_model import ScriptedModel
from types_mod import ModelTurn, ToolCall
from loop import run


def script_happy_path() -> list[ModelTurn]:
    """Turn 1: read app.py. Turn 2: report done."""
    return [
        ModelTurn("", (ToolCall("c1", "read_file", {"path": "app.py"}),), "tool_use"),
        ModelTurn("app.py says hello.", (), "end_turn"),
    ]


def script_unknown_tool() -> list[ModelTurn]:
    """Model asks for a tool that doesn't exist, then recovers to echo."""
    return [
        ModelTurn("", (ToolCall("c1", "list_dir", {"path": "."}),), "tool_use"),
        ModelTurn("", (ToolCall("c2", "echo", {"message": "falling back"}),), "tool_use"),
        ModelTurn("Recovered.", (), "end_turn"),
    ]


def script_bad_schema() -> list[ModelTurn]:
    """Model passes a typo'd argument, then corrects it."""
    return [
        ModelTurn("", (ToolCall("c1", "read_file", {"pth": "app.py"}),), "tool_use"),
        ModelTurn("", (ToolCall("c2", "read_file", {"path": "app.py"}),), "tool_use"),
        ModelTurn("Got it.", (), "end_turn"),
    ]


def script_tool_exception() -> list[ModelTurn]:
    """Model reads a missing file — a FileNotFoundError becomes a tool result."""
    return [
        ModelTurn("", (ToolCall("c1", "read_file", {"path": "nope.py"}),), "tool_use"),
        ModelTurn("No such file.", (), "end_turn"),
    ]


def script_path_escape() -> list[ModelTurn]:
    """Model asks for a path outside the workspace; the tool raises PermissionError."""
    return [
        ModelTurn("", (ToolCall("c1", "read_file", {"path": "../../etc/passwd"}),), "tool_use"),
        ModelTurn("Refused.", (), "end_turn"),
    ]


def summarize(name: str, result) -> None:
    print(f"\n--- {name} ---")
    print(f"stop_reason={result.stop_reason}, turns={result.turns}, text={result.final_text!r}")
    for m in result.transcript:
        if m.get("kind") == "tool_call":
            c = m["call"]
            print(f"  → call {c.name}({dict(c.input)})")
        elif m.get("kind") == "tool_result":
            r = m["result"]
            tag = "ERR" if r.is_error else "ok "
            print(f"  ← {tag} {r.content[:80]}")


if __name__ == "__main__":
    for name, script_fn in [
        ("happy path", script_happy_path),
        ("unknown tool", script_unknown_tool),
        ("bad schema", script_bad_schema),
        ("tool exception", script_tool_exception),
        ("path escape", script_path_escape),
    ]:
        summarize(name, run(ScriptedModel(script_fn()), task="read app.py"))
```

Run it:

```bash
python3 drive.py
```

**Expected output (abbreviated):**

```
--- happy path ---
stop_reason=done, turns=2, text='app.py says hello.'
  → call read_file({'path': 'app.py'})
  ← ok  hello from app.py

--- unknown tool ---
stop_reason=done, turns=3, text='Recovered.'
  → call list_dir({'path': '.'})
  ← ERR invalid tool call: unknown tool 'list_dir'; available: ['echo', 'read_file']
  → call echo({'message': 'falling back'})
  ← ok  falling back

--- bad schema ---
stop_reason=done, turns=3, text='Got it.'
  → call read_file({'pth': 'app.py'})
  ← ERR invalid tool call: missing required args: ['path']
  → call read_file({'path': 'app.py'})
  ← ok  hello from app.py

--- tool exception ---
stop_reason=done, turns=2, text='No such file.'
  → call read_file({'path': 'nope.py'})
  ← ERR FileNotFoundError: [Errno 2] No such file or directory: ...

--- path escape ---
stop_reason=done, turns=2, text='Refused.'
  → call read_file({'path': '../../etc/passwd'})
  ← ERR PermissionError: '../../etc/passwd' is outside the workspace
```

Five things to notice:

- **Every scenario ends with `stop_reason=done`.** The loop didn't crash on any of them; Chapter 2's *"the loop only stops on purpose"* is literal.
- **The error messages tell the model what to do next.** Unknown tool → the real names. Bad schema → the missing argument's name. Path escape → *why* it was refused. Chapter 2: *"Bad input gets the schema. A denial gets the reason."*
- **Each tool call is followed by exactly one tool result.** No exceptions — including the three error paths. Chapter 2: *"Every tool call gets a result."*
- **The transcript is append-only.** Every script above produces the same transcript on every run, which is what makes it a regression test seed for Exercise 01.
- **The "list_dir" scenario is realistic.** Models trained on general tool catalogs regularly try `list_dir`, `ls`, or `cd` when they haven't seen your registry; the error message lets them recover in one turn rather than guess for three.

## Step 7 — Break the "every call gets a result" invariant

Chapter 2 claims provider APIs reject a mismatched history. Prove it with a 20-line "provider check." Add to `drive.py` (or save as `check_pairs.py`):

```python
def check_tool_call_pairs(history: list) -> list[str]:
    """Return a list of invariant violations, or [] if the history is well-formed."""
    problems = []
    pending: dict[str, str] = {}
    for i, m in enumerate(history):
        if m.get("kind") == "tool_call":
            cid = m["call"].id
            if cid in pending:
                problems.append(f"index {i}: duplicate tool_call id {cid!r}")
            pending[cid] = m["call"].name
        elif m.get("kind") == "tool_result":
            rid = m["result"].call_id
            if rid not in pending:
                problems.append(f"index {i}: tool_result {rid!r} without a matching call")
            else:
                del pending[rid]
    for cid, name in pending.items():
        problems.append(f"call {cid!r} ({name}) has no tool_result")
    return problems


# Run the happy-path scenario, then deliberately drop the tool result:
result = run(ScriptedModel(script_happy_path()), task="read app.py")
print("well-formed:", check_tool_call_pairs(list(result.transcript)))

broken = [m for m in result.transcript if m.get("kind") != "tool_result"]
print("after dropping the result:", check_tool_call_pairs(broken))
```

**Expected output:**

```
well-formed: []
after dropping the result: ["call 'c1' (read_file) has no tool_result"]
```

This is the exact shape the Anthropic Messages API rejects. The point of Chapter 2's "every tool call gets a result" rule is that a loop which *skips* appending a result on error — a common temptation — breaks the provider API before it reaches the model's reasoning. Carry the check forward into Exercise 01's unit tests.

## Step 8 — Watch loop detection fire

The scripted model has one more job: simulating a stuck agent. Add one more script:

```python
def script_stuck() -> list[ModelTurn]:
    """Model asks for the same read three times in a row."""
    same_call = ToolCall("c1", "read_file", {"path": "app.py"})
    return [
        ModelTurn("", (same_call,), "tool_use"),
        ModelTurn("", (ToolCall("c2", "read_file", {"path": "app.py"}),), "tool_use"),
        ModelTurn("", (ToolCall("c3", "read_file", {"path": "app.py"}),), "tool_use"),
        ModelTurn("Done.", (), "end_turn"),  # never reached
    ]


summarize("stuck", run(ScriptedModel(script_stuck()), task="read app.py"))
```

**Expected output:**

```
--- stuck ---
stop_reason=loop_detected, turns=3, text=''
  → call read_file({'path': 'app.py'})
  ← ok  hello from app.py
  → call read_file({'path': 'app.py'})
  ← ok  hello from app.py
  → call read_file({'path': 'app.py'})
  ← ok  hello from app.py
```

Three things to notice, from Chapter 2:

- **The script offers a fourth turn, but the loop never runs it.** `stop_reason=loop_detected` is reached before `complete()` is called a fourth time. Chapter 2: *"The model is stuck calling the same tool with the same input. Stop instead of paying for more of it."*
- **The stop reason is distinct from `done`.** Chapter 2: *"'Finished' and 'ran out of turns' look the same if all you keep is the final text, and they mean very different things."* Return the reason with the result — the CLI from Chapter 8 branches exit codes on it.
- **Loop detection sits outside the model.** The model doesn't know it's stuck; the harness decides. A guide in the system prompt saying "don't repeat yourself" is a hint; the stop condition is a bound.

## Step 9 — Checkpoint

Without peeking, you can state:

1. The four stop conditions from Chapter 2, and which function in `loop.py` holds each one.
2. Why the schema-validator and the `try/except` around the tool function return `ToolResult(is_error=True)` instead of raising — in one sentence that would fit above `execute()`.
3. Why `_recent_calls` keys loop detection on `(name, sorted-input)` rather than on the `ToolCall.id` — and what would break if it used `id`.

## Wrap-up

Carry the four types, the registry shape, the dispatch function, and the pair-check invariant into [Exercise 01](../exercises/exercise-01-build-a-minimal-harness.md), which asks you to:

- Replace the two pure-function tools with `read_file`, `write_file`, `list_dir`, and `run_command` against a real workspace, with `shlex`-based argv parsing, program allowlists, timeouts, and head-and-tail truncation.
- Insert a `PermissionPolicy` (allow / deny / ask with an approver callback) and `pre_tool`/`post_tool` hook stages into `execute()`.
- Add a token budget as a fourth stop condition alongside `loop_detected`, `max_turns`, and `done`.
- Swap `ScriptedModel` for an `AnthropicModel` for one live run, keeping every test on the scripted path.

What this lab closes: the *shape* of the loop and its invariants (errors as observations, every call paired with a result, append-only history, stop reasons that aren't "done") are no longer a mystery. Exercise 01 is now a question of what goes inside each slot, not reverse-engineering what the slots are.

### Stretch

- Add a token-budget stop condition using `len(str(history)) // 4` as the estimate, and prove with a script that it fires before `max_turns` on a long tool-result turn. Chapter 5 revisits the estimator.
- Add a `permission` stage between `validate` and the `try/except` block: a pure function `(ToolCall) -> "allow" | "deny" | "ask"`. Prove with a script that a `deny` on `read_file` returns an error result rather than crashing, and that the model's next turn can recover with `echo`.
- Turn the scripted model into a recorder: a wrapper that logs every `(messages, reply)` pair to a cassette file, so Exercise 06's replay tests have a starter format.
