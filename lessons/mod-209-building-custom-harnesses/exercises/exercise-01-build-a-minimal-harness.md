# exercise-01: Build a Minimal Harness

**Estimated effort:** 4 hours

## Objective

Write your own coding-agent harness from scratch: the loop, a tool registry, a permission gate, pre- and post-tool hooks, and stop conditions for turns and budget. Keep it small enough to read in one sitting (a few hundred lines). By the end you'll understand what Claude Code and Codex are doing under the hood, because you'll have built a working version of it.

## Background

This exercise covers material from:

- [Chapter 1 — Harness Architecture](../01-harness-architecture.md)
- [Chapter 2 — The Agent Loop](../02-the-agent-loop.md)

In [mod-201](../../mod-201-agent-fundamentals/README.md) you wrote a reason-act loop. That loop was the agent. This exercise treats the loop as one part of a harness and adds the parts around it that make it safe to point at a real filesystem.

## Prerequisites

- Python 3.11+, `pytest`.
- The `anthropic` SDK for live runs (any provider with tool calling works). All tests use a scripted fake model.
- A spend cap for the live run.

## Tasks

### 1. Model client interface

- Define a `ModelClient` protocol: `complete(system, messages, tools) -> ModelTurn`, where `ModelTurn` has `text`, `tool_calls` (list of `id`, `name`, `input`), `stop_reason`, and `usage` (`input_tokens`, `output_tokens`).
- Implement `ScriptedModel` for tests: it returns a fixed sequence of turns you specify.
- Implement `AnthropicModel` that maps the Messages API to this interface.

### 2. Tool registry

- Tools are plain Python functions registered with a name, description, and JSON Schema for their input.
- Provide four: `read_file`, `write_file`, `list_dir`, and `run_command`. All paths resolve inside a workspace root; reject anything that escapes it (`../`, absolute paths outside, symlinks pointing out).
- `run_command` has a timeout and truncates output to a fixed size, keeping the head and tail.

### 3. The loop

- Send system prompt, history, and tool schemas. If the model returns tool calls, execute each and append the results; if it returns no tool calls, finish.
- Tool errors (bad input, exception, timeout) become tool results marked as errors that the model can read. They must not crash the loop.
- Validate tool input against its schema before calling the function.

### 4. Permission gate

- Before each tool call, ask a `PermissionPolicy`: `allow`, `deny`, or `ask`.
- Provide a rule-based policy (for example: reads always allowed; writes allowed inside `src/` and `tests/`; `run_command` allowed for an allowlist of programs; everything else `ask`).
- `ask` calls an approver callback. In tests it's a stub; in the CLI it prompts on the terminal.
- A denial becomes a tool result explaining why, so the model can adjust.

### 5. Hooks

- Support `pre_tool` and `post_tool` hooks as a list of callables. A `pre_tool` hook can block a call with a reason. A `post_tool` hook can append a note to the tool result (for example, lint output after a write).
- Port one rule from your [mod-208 exercise-02](../../mod-208-harness-engineering/exercises/exercise-02-guardrail-hooks.md) guard as a `pre_tool` hook.

### 6. Stop conditions

- Stop on: no tool calls (done), `max_turns` reached, token budget exceeded, or repeated identical tool calls (the same name and input three times in a row).
- Return a `RunResult` with the final text, the stop reason, turns used, total tokens, and the full transcript.

## Starter guidance

```python
from dataclasses import dataclass, field
from typing import Any, Callable, Literal, Protocol

Decision = Literal["allow", "deny", "ask"]

@dataclass(frozen=True)
class ToolCall:
    id: str
    name: str
    input: dict[str, Any]

@dataclass(frozen=True)
class ModelTurn:
    text: str
    tool_calls: tuple[ToolCall, ...]
    stop_reason: str
    input_tokens: int
    output_tokens: int

class ModelClient(Protocol):
    def complete(self, system: str, messages: list[dict], tools: list[dict]) -> ModelTurn: ...

@dataclass(frozen=True)
class RunResult:
    final_text: str
    stop_reason: Literal["done", "max_turns", "budget", "loop_detected"]
    turns: int
    total_tokens: int
    transcript: tuple[dict, ...] = field(default_factory=tuple)
```

## Acceptance criteria

You can demonstrate that, with `ScriptedModel` in `pytest`:

- A scripted two-step task (list a directory, then read a file) finishes with `stop_reason == "done"`.
- Path escapes (`../../etc/passwd`, an absolute path outside the workspace, a symlink out) are rejected as tool errors, not executed.
- A tool exception and a schema-invalid input each come back to the model as error results, and the loop continues.
- `deny` and a blocking `pre_tool` hook each stop the call and tell the model why; `ask` calls the approver.
- Each stop condition triggers: `max_turns`, `budget`, and `loop_detected`.
- One live run against a real model completes a small task in a scratch workspace.

## Reflection

In `NOTES.md`:

1. Which part of your harness did you most underestimate? What does that suggest about the harnesses you use every day?
2. Your path check and command allowlist run in the same process as the agent. What can a determined model still do, and what's the real isolation boundary? (Exercise-02 comes back to this.)
3. Compare your loop with Claude Code or Codex on the same task. What does the production harness do that yours doesn't?

## Stretch goals

- Add parallel execution for independent read-only tool calls in the same turn.
- Run `run_command` inside a container or a sandbox (`bwrap`, `sandbox-exec`, or Docker) instead of the host.
- Add a `subagent` tool that runs a nested loop with its own fresh history and a restricted tool set, and returns only a summary.
