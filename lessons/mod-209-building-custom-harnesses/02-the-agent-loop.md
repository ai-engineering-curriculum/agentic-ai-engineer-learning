# Chapter 2 — The Agent Loop

[Chapter 1](01-harness-architecture.md) laid out the parts of a harness and how they fit together. This chapter builds the core: the loop that calls the model, dispatches tools, and decides when to stop. Everything later in the module plugs into it.

## The shape of a harness

```text
            ┌──────────────── harness ─────────────────┐
 task ──▶   │  system prompt + history + tool schemas   │
            │              │                            │
            │              ▼                            │
            │         model.complete() ◀──────────┐     │
            │              │                      │     │
            │     tool calls? ──no──▶ done        │     │
            │              │yes                   │     │
            │   for each call:                    │     │
            │     validate input                  │     │
            │     pre_tool hooks (may block)      │     │
            │     permission gate (allow/deny/ask)│     │
            │     execute (sandboxed, timed out)  │     │
            │     post_tool hooks (may annotate)  │     │
            │     append result ──────────────────┘     │
            │   check stop conditions                   │
            └───────────────────────────────────────────┘
```

The model sits in the middle of the loop, but most of the code is the harness around it.

## The loop

```python
def run(task: str) -> RunResult:
    history = [user(task)]
    for turn in range(1, MAX_TURNS + 1):
        reply = model.complete(SYSTEM, history, registry.schemas())
        history = [*history, assistant(reply)]
        if not reply.tool_calls:
            return RunResult(reply.text, "done", turn, tokens(history), tuple(history))
        results = [execute(call) for call in reply.tool_calls]
        history = [*history, tool_results(results)]
        if over_budget(history):
            return RunResult("", "budget", turn, tokens(history), tuple(history))
        if repeating(history):
            return RunResult("", "loop_detected", turn, tokens(history), tuple(history))
    return RunResult("", "max_turns", MAX_TURNS, tokens(history), tuple(history))
```

Three choices in this small function matter:

- **No tool calls means done.** The model signals it has finished by answering without asking for a tool. Don't look for magic words in the text.
- **Every tool call gets a result.** The provider APIs require each tool call to be paired with a result in the next message. If a call is denied or fails, it still gets a result; the result just says so.
- **History is rebuilt, not mutated.** Building a new list each turn makes the transcript easy to snapshot, replay, and test.

## Tool dispatch

A tool registry maps names to functions, descriptions, and input schemas. Dispatch is where most harness bugs live, because every call carries input chosen by a model:

```python
def execute(call: ToolCall) -> ToolResult:
    tool = registry.get(call.name)
    if tool is None:
        return error(call, f"Unknown tool {call.name!r}. Available: {registry.names()}")
    problems = validate(call.input, tool.schema)
    if problems:
        return error(call, f"Invalid input: {problems}. Schema: {tool.schema}")
    blocked = first_block(pre_hooks, call)
    if blocked:
        return error(call, blocked)
    decision = policy.decide(call)
    if decision == "ask":
        decision = "allow" if approver(call) else "deny"
    if decision == "deny":
        return error(call, policy.reason(call))
    try:
        output = tool.fn(**call.input)
    except Exception as exc:
        return error(call, f"{type(exc).__name__}: {exc}")
    return annotate(post_hooks, call, ok(call, output))
```

**Errors are observations.** Every failure path returns a result the model can read and react to. An unknown tool name gets the list of real ones. Bad input gets the schema. A denial gets the reason. An exception gets its message. A loop that crashes on bad tool input breaks the first time a model misspells an argument. A loop that returns the error lets the model correct itself on the next turn.

## Path containment

Any file tool must keep the agent inside its workspace. The naive check, `path.startswith(root)`, fails on `../`, on prefixes (`/work` vs. `/workspace-other`), and on symlinks. Resolve first, then compare:

```python
def contained(root: Path, requested: str) -> Path:
    candidate = (root / requested).resolve()
    if not candidate.is_relative_to(root.resolve()):
        raise PermissionError(f"{requested!r} is outside the workspace")
    return candidate
```

`resolve()` follows symlinks and collapses `..`, so a link inside the workspace that points to `/etc` resolves to `/etc` and fails the check. This is still an in-process check: a `run_command` tool can step around it. The real boundary is the OS sandbox from [Chapter 4](04-execution-isolation.md). Do both.

## Running commands

A `run_command` tool is the most powerful tool and the riskiest. Minimum safeguards:

- **No shell.** Split with `shlex.split` and run the argument list with `subprocess.run(..., shell=False)`. This blocks the easy injection tricks (`;`, `&&`, `$(...)`) and makes an allowlist meaningful.
- **An allowlist of programs** (`pytest`, `ruff`, `git`, `ls`), checked against the first argument.
- **A timeout** on every call, with the timeout reported to the model as an error.
- **Truncated output, head and tail.** Errors are usually at the end of output and context at the start. Keep both and note how much was cut.

## Permissions and hooks as middleware

The permission gate and hooks are middleware around the tool call. Keep them as plain callables the loop calls in order, not logic woven into each tool:

- **Policy** answers allow, deny, or ask, based on the tool and its input. A rule list ("reads allowed; writes allowed under `src/` and `tests/`; commands allowed if on the allowlist; otherwise ask") covers most needs.
- **Approver** is a callback for `ask`. In a CLI it prompts on the terminal. In tests it's a stub. In an unattended pipeline it's "deny", since no human is there.
- **Pre-tool hooks** inspect the call and may block it with a reason. This is where the guard from mod-208's exercise-02 goes.
- **Post-tool hooks** inspect the result and may add to it: run the linter after a write and append its output, so the model sees the problem in the same turn.

Keeping these separate from the tools means you can test each piece alone and change policy without touching tool code.

**Order matters.** The sketch above runs pre-tool hooks before the permission gate, as Claude Code does: a hook can block a call outright, but it can't approve one the policy would deny. Running the gate first works too. Pick one order, document it, and test it, because "which check fired?" is the first question you'll ask when a call is blocked unexpectedly. A hook that crashes should block the call (fail closed). A post-tool hook that crashes should only add a note (fail open), since the tool has already run.

## Stop conditions

An agent loop must always end. Four conditions cover most cases:

| Condition | Why |
|---|---|
| No tool calls | The model says it's done. |
| `max_turns` | A ceiling on any run, whatever happens. |
| Token budget | Cost control; long runs can cost far more than you expect. |
| Repeated identical calls | The model is stuck calling the same tool with the same input. Stop instead of paying for more of it. |

Return the stop reason with the result. "Finished" and "ran out of turns" look the same if all you keep is the final text, and they mean very different things.

## Key takeaways

- A harness is mostly code around the model: validation, permissions, hooks, execution, and stop conditions.
- No tool calls means done. Every tool call gets a result, including denials and errors.
- Errors are observations: return them to the model with enough detail to correct course.
- Resolve paths before checking containment. Run commands without a shell, with an allowlist, a timeout, and head-and-tail truncation.
- Keep permissions and hooks as separate middleware so each can be tested and changed alone.
- Always have a ceiling: max turns, token budget, and loop detection, and report which one ended the run.
