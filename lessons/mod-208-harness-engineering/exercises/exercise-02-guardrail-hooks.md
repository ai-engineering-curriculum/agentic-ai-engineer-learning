# exercise-02: Deterministic Guardrails with Hooks

**Estimated effort:** 3 hours

## Objective

Write three hook scripts that enforce rules the agent can't talk its way around: block dangerous tool calls before they run, format files after they're edited, and refuse to let the agent stop while checks are failing. Test each hook offline by feeding it the same JSON the harness sends. By the end you'll have hooks you trust because they have tests, not because they seemed to work once.

## Background

This exercise covers material from:

- [Chapter 3 — Tools, Permissions, and Hooks](../03-tools-permissions-hooks.md)

The exercise targets Claude Code's hook contract: the hook receives a JSON event on stdin, exit code `0` means allow, exit code `2` means block (stderr is shown to the model), and any other code is a non-blocking error. Codex and other agents have similar mechanisms; if you use one of them, keep the same three behaviors and adapt the wire format.

## Prerequisites

- Python 3.11+ and `pytest`.
- Claude Code, for the final live check (task 5). Tasks 1–4 run offline.
- [exercise-01](exercise-01-harness-a-repository.md) helps but isn't required; any repo with a check command works.

## Tasks

### 1. `pre_tool_guard.py` (PreToolUse)

Block, with exit code `2` and a clear stderr message the model can act on:

- `Bash` commands matching destructive patterns: `rm -rf /`, `rm -rf ~`, `git push --force` to `main`/`master`, and `curl ... | sh`.
- `Write` or `Edit` to protected paths: `.env*`, anything under `.git/`, `secrets/`, and lockfiles (`uv.lock`, `package-lock.json`).
- Anything else passes with exit `0`.

Load patterns and paths from a small config (a dict or a JSON file), not from code scattered across `if` statements.

### 2. `post_edit_format.py` (PostToolUse)

- After `Write` or `Edit` on a `.py` file, run the formatter (`ruff format <file>`) on that one file.
- If the formatter isn't installed or fails, print a warning to stderr and exit `0`. A broken formatter shouldn't block work.
- Never format files outside the project directory, even if the event points there.

### 3. `stop_gate.py` (Stop)

- Run the project's check command (default `make check`, overridable by an environment variable).
- If it fails, block the stop: output JSON `{"decision": "block", "reason": "<last 40 lines of output>"}` so the agent keeps working on the failure.
- Guard against an infinite loop: if the event has `stop_hook_active: true`, allow the stop. The agent already had one chance to fix it.

### 4. Tests

- For each hook, write `pytest` tests that run the script as a subprocess, pipe a JSON event to stdin, and assert on exit code and stderr/stdout.
- Cover at least: every block rule, a near-miss that must pass (`rm -rf ./build`, `git push origin feature`), a malformed JSON event (must not crash with a traceback; fail closed for the guard, fail open for the formatter), and the `stop_hook_active` loop guard.
- Use a fake check command in the stop-gate tests so they don't depend on a real project.

### 5. Wire and verify live

- Register all three in `.claude/settings.json` with the right matchers.
- In a live session, ask the agent to do something the guard blocks and confirm it sees the reason and changes course. Break a test, ask the agent to finish, and confirm the stop gate sends it back.

## Starter guidance

```python
#!/usr/bin/env python3
"""PreToolUse guard: exit 2 blocks the call and shows stderr to the model."""
import json
import re
import sys

BLOCKED_BASH = [
    (r"\brm\s+-rf\s+(/|~)(\s|$)", "Refusing to delete the filesystem root or home directory."),
    (r"\bgit\s+push\b.*--force.*\b(main|master)\b", "Force-pushing to the default branch is not allowed."),
    (r"curl[^|]*\|\s*(sh|bash)\b", "Piping a download into a shell is not allowed."),
]

def main() -> int:
    try:
        event = json.load(sys.stdin)
    except json.JSONDecodeError:
        print("guard: could not parse hook input; blocking to be safe", file=sys.stderr)
        return 2
    tool = event.get("tool_name")
    tool_input = event.get("tool_input", {})
    # TODO: check Bash commands against BLOCKED_BASH
    # TODO: check Write/Edit file_path against protected paths
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

```json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Bash|Write|Edit",
        "hooks": [{ "type": "command", "command": "python3 .claude/hooks/pre_tool_guard.py" }] }
    ]
  }
}
```

Check the current hook event names, input fields, and output format in the official docs before wiring. The contract changes between versions.

## Acceptance criteria

You can demonstrate that:

- `pytest` passes, with tests for every block rule, at least two near-misses, malformed input, and the stop-hook loop guard.
- The guard fails closed on bad input (exit `2`); the formatter fails open (exit `0` with a warning).
- Block messages tell the model what to do instead, not just "denied".
- In a live session, the agent hits a block, reads the reason, and picks a different approach.
- The stop gate sends the agent back once on a failing check and lets it stop when `stop_hook_active` is set.

## Reflection

In `NOTES.md`:

1. Why is a hook stronger than the same rule written in `AGENTS.md`? When is the instruction file still the right place for a rule?
2. A regex guard on shell commands can be bypassed (`r''m -rf /`, a script file, `python -c`). What's the backstop? (Hint: Chapter 3's section on sandboxing.)
3. The stop gate makes sessions longer and more expensive. When is it worth it, and when would you turn it off?

## Stretch goals

- Make the guard emit structured JSON (`permissionDecision: "deny"` plus a reason) instead of relying on exit code `2`, and test both paths.
- Add a `SessionStart` hook that injects the current branch, the last three commits, and failing tests into context.
- Log every hook decision to a JSONL file and summarize which rules fire most over a week.
