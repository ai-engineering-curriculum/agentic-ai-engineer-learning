# Lab 2 — Pipe Hook Events to Feel the Contract

**Duration:** ~45 minutes
**Covers:** [Chapter 3 — Tools, Permissions, and Hooks](../03-tools-permissions-hooks.md)

## Objective

Chapter 3 says a hook is a program the harness runs at a lifecycle point: it gets a JSON event on stdin, decides what happens via its exit code (and optionally structured JSON on stdout), and whatever it writes to stderr on `exit 2` is shown to the model as the reason. Before [Exercise 02](../exercises/exercise-02-guardrail-hooks.md) asks you to build three production hooks with a `pytest` suite and wire them into a live session, spend 45 minutes driving the **smallest possible** `PreToolUse` hook from the shell so the wire format itself is visible — no live agent, no tokens.

By the end of the lab you will:

- Write a ~25-line Python `PreToolUse` hook that reads one event from stdin and decides allow/deny.
- Pipe four hand-crafted events through it from Bash — an allowed edit, a denied edit, an allowed Bash call, a denied Bash call — and watch the exit code and stderr.
- Feel why Chapter 3 is specific about block messages: swap a bare `"denied"` message for one that tells the model what to do instead, and see the difference in what a next agent attempt would receive.
- Switch the hook from "exit 2 + stderr" to the newer "stdout JSON with `permissionDecision: deny`" shape and see both paths carry the same information.

Pure stdlib; no agent; no pytest; no `.claude/settings.json` wiring. Those additions are what Exercise 02 adds on top.

## Prerequisites

- Python 3.11+.
- Bash, `jq` (optional but convenient).
- [Chapter 3](../03-tools-permissions-hooks.md) read, especially the "Hooks: deterministic code at lifecycle points" section.

## Setup

```bash
mkdir lab-02 && cd lab-02
```

No venv needed — the hook uses only `json`, `sys`, `re`, and `fnmatch`.

## Step 1 — The smallest `PreToolUse` guard

The hook's contract is very small: JSON on stdin, decision via exit code. The deliberate design is "a program, with your permissions, that runs no matter what the model decides."

Save as `pre_tool_guard.py` and make it executable (`chmod +x pre_tool_guard.py`):

```python
#!/usr/bin/env python3
"""PreToolUse guard: exit 0 allows, exit 2 blocks with stderr shown to the model."""
import json
import re
import sys
from fnmatch import fnmatch


PROTECTED_PATHS = [".env*", ".git/*", "*.lock", "secrets/*"]

BLOCKED_BASH = [
    (re.compile(r"\brm\s+-rf\s+(/|~)(\s|$)"),
     "Refusing to delete the filesystem root or home directory."),
    (re.compile(r"\bgit\s+push\b.*--force.*\b(main|master)\b"),
     "Force-pushing to the default branch is not allowed. Open a PR instead."),
]


def _matches_protected(path: str) -> bool:
    return any(fnmatch(path, p) or fnmatch(path, f"*/{p}") for p in PROTECTED_PATHS)


def decide(event: dict) -> tuple[int, str]:
    tool = event.get("tool_name")
    tin = event.get("tool_input", {}) or {}
    if tool in {"Write", "Edit"}:
        path = tin.get("file_path", "")
        if _matches_protected(path):
            return 2, (
                f"{path} is protected. Lockfiles: change the manifest and rerun the "
                "package manager. Secrets/env: change the deploy config, never the file."
            )
    if tool == "Bash":
        cmd = tin.get("command", "")
        for pattern, reason in BLOCKED_BASH:
            if pattern.search(cmd):
                return 2, reason
    return 0, ""


def main() -> int:
    try:
        event = json.load(sys.stdin)
    except json.JSONDecodeError:
        print("guard: could not parse hook input; blocking to be safe", file=sys.stderr)
        return 2
    code, reason = decide(event)
    if code == 2:
        print(reason, file=sys.stderr)
    return code


if __name__ == "__main__":
    sys.exit(main())
```

Three details worth noticing:

- **`decide` is a pure function** of the event dict. The I/O is at the edges (stdin / stderr / exit code). That's what lets the unit tests in Exercise 02 be fast and deterministic.
- **Fail closed on malformed input.** A `JSONDecodeError` returns exit 2, not exit 0 — Chapter 3's point that a guard whose job is *to block* must not silently let traffic through when it can't parse the request.
- **Block messages explain the alternative.** Chapter 3: *"A bare 'denied' leads to the agent retrying slight variations."* Both reasons in `BLOCKED_BASH` and the protected-path message name what to do instead.

## Step 2 — Four hand-crafted events

Hooks don't know where the JSON on their stdin came from — that's what makes them scriptable from a shell. Save these as four files so the next step's pipe commands stay readable.

```bash
cat > ev_allow_edit.json <<'JSON'
{"tool_name": "Edit", "tool_input": {"file_path": "src/tool/cli.py"}}
JSON

cat > ev_deny_lockfile.json <<'JSON'
{"tool_name": "Edit", "tool_input": {"file_path": "uv.lock"}}
JSON

cat > ev_allow_bash.json <<'JSON'
{"tool_name": "Bash", "tool_input": {"command": "pytest tests/"}}
JSON

cat > ev_deny_bash.json <<'JSON'
{"tool_name": "Bash", "tool_input": {"command": "git push --force origin main"}}
JSON
```

The field names (`tool_name`, `tool_input.file_path`, `tool_input.command`) are the live wire format at the time of writing; check the official docs before you ship a real hook, because this contract changes between harness versions.

<!-- needs-research: confirm current PreToolUse field names across latest Claude Code and Codex versions before students lift this hook into production. -->

## Step 3 — Drive the hook from the shell

Pipe each event through the hook and inspect the exit code:

```bash
cat ev_allow_edit.json | python3 pre_tool_guard.py ; echo "exit=$?"
cat ev_deny_lockfile.json | python3 pre_tool_guard.py ; echo "exit=$?"
cat ev_allow_bash.json | python3 pre_tool_guard.py ; echo "exit=$?"
cat ev_deny_bash.json | python3 pre_tool_guard.py ; echo "exit=$?"
```

**Expected output:**

```
exit=0
uv.lock is protected. Lockfiles: change the manifest and rerun the package manager. Secrets/env: change the deploy config, never the file.
exit=2
exit=0
Force-pushing to the default branch is not allowed. Open a PR instead.
exit=2
```

Three things to notice:

- **Exit 0 is silent.** The harness continues; the model never learns the hook even ran. Chapter 3: *"Allow the things the agent needs to do constantly and that are safe to repeat... every unnecessary prompt slows the human down."*
- **Exit 2 prints the reason on stderr.** That stderr text is what the model sees as the block reason — the entire reason it should pick a different action next turn.
- **Near-misses pass.** `rm -rf ./build` and `git push origin feature` would both exit 0; the chapter's own example, and a deliberate narrowness. A guard that denies everything close to dangerous teaches the human to approve anything.

Exercise one of the near-misses to confirm:

```bash
echo '{"tool_name": "Bash", "tool_input": {"command": "rm -rf ./build"}}' | python3 pre_tool_guard.py ; echo "exit=$?"
```

**Expected:** `exit=0`.

## Step 4 — Feel why block messages matter

Chapter 3 is specific that block messages should say what to do instead. Temporarily weaken the message to see what the old-school "deny" costs you. Edit the lockfile branch of `decide` to say only `"denied"`:

```python
if _matches_protected(path):
    return 2, "denied"
```

Re-run:

```bash
cat ev_deny_lockfile.json | python3 pre_tool_guard.py ; echo "exit=$?"
```

**Expected:**

```
denied
exit=2
```

Now pretend you are the model receiving that stderr. You asked to edit `uv.lock`. You now know the harness refused. You do not know:

- Whether the refusal is about the *path* (`uv.lock`), or the *tool* (`Edit`), or your *content* (what you tried to write).
- Whether retrying with a slightly different approach (`sed -i` through Bash? A `Write` instead of an `Edit`?) is the fix.
- Whether lockfiles should be regenerated via a command the harness is happy to run.

The chapter's point: without the "do this instead" sentence, the agent's next guess is a lottery, and probably costs you a wasted turn. Revert the change before continuing:

```python
if _matches_protected(path):
    return 2, (
        f"{path} is protected. Lockfiles: change the manifest and rerun the "
        "package manager. Secrets/env: change the deploy config, never the file."
    )
```

## Step 5 — The structured-JSON output path

The chapter mentions that hooks can print structured JSON for finer control. Add a flag to the hook so it emits a JSON decision on stdout instead of using exit code 2:

```python
# add near the top:
import os
USE_JSON = os.environ.get("GUARD_JSON") == "1"

# replace main():
def main() -> int:
    try:
        event = json.load(sys.stdin)
    except json.JSONDecodeError:
        if USE_JSON:
            print(json.dumps({"permissionDecision": "deny",
                              "reason": "could not parse hook input"}))
            return 0
        print("guard: could not parse hook input; blocking to be safe", file=sys.stderr)
        return 2
    code, reason = decide(event)
    if USE_JSON:
        decision = {"permissionDecision": "deny", "reason": reason} if code == 2 \
            else {"permissionDecision": "allow"}
        print(json.dumps(decision))
        return 0
    if code == 2:
        print(reason, file=sys.stderr)
    return code
```

Run the same four events under the JSON path:

```bash
for ev in ev_allow_edit.json ev_deny_lockfile.json ev_allow_bash.json ev_deny_bash.json; do
  echo "--- $ev ---"
  GUARD_JSON=1 cat "$ev" | GUARD_JSON=1 python3 pre_tool_guard.py ; echo "exit=$?"
done
```

**Expected output:**

```
--- ev_allow_edit.json ---
{"permissionDecision": "allow"}
exit=0
--- ev_deny_lockfile.json ---
{"permissionDecision": "deny", "reason": "uv.lock is protected. Lockfiles: ..."}
exit=0
--- ev_allow_bash.json ---
{"permissionDecision": "allow"}
exit=0
--- ev_deny_bash.json ---
{"permissionDecision": "deny", "reason": "Force-pushing to the default branch..."}
exit=0
```

Two things to notice:

- **Same decision, different channel.** The *information* is identical: allow or deny, plus the reason on deny. The exit-code-and-stderr path and the JSON path are two ways to carry the same contract.
- **The JSON path returns exit 0 even on deny.** The decision is in the payload, not the exit code, so the harness won't confuse a parse error (still exit 2) with a deliberate deny (exit 0, JSON says deny). Exercise 02's stretch goal asks you to test both paths.

<!-- needs-research: confirm the exact JSON shape (field name "permissionDecision" vs "decision", extra required fields) against current Claude Code docs before wiring live. -->

## Step 6 — Checkpoint

Without peeking, you can state:

1. The three things a `PreToolUse` hook returns to the harness (exit code, stderr, optional JSON on stdout) and which one the model sees on a deny.
2. Why Chapter 3 wants the block message to explain the alternative — in one sentence that would fit above a `BLOCKED_BASH` entry.
3. Why the hook fails closed on malformed JSON instead of allowing through — and which other hook from Exercise 02 is specifically allowed to fail *open* and why.

## Wrap-up

Carry the hook and the four events into [Exercise 02](../exercises/exercise-02-guardrail-hooks.md), which asks you to:

- Add a `PostToolUse` formatter hook (`ruff format` on the edited file) that fails open, not closed.
- Add a `Stop` gate that runs `make check` and blocks the stop on failure, respecting `stop_hook_active` to avoid infinite loops.
- Write `pytest` tests that pipe JSON events as subprocesses and assert on exit code and stderr for every rule, including malformed input and near-misses.
- Wire all three into `.claude/settings.json` and verify live.

What this lab closes: the *wire format* is no longer a mystery. Exercise 02 is now a question of writing good rules and testing them, not reverse-engineering how hooks talk to the harness.

### Stretch

- Load `PROTECTED_PATHS` and `BLOCKED_BASH` from a `guard_config.json` file. A real guard's rules change often; Chapter 3 recommends loading them from config, not from `if` statements scattered through the code.
- Add a `tool_name == "Read"` branch that blocks reads of `secrets/*` and `.env*` — reading a secret into context is as leaky as writing to it in some threat models.
- Add a `SessionStart` event handler: a different script that *injects* text into the session (current branch, last 3 commits, failing tests) via stdout. Chapter 3's `SessionStart` row of the event table; a different half of the contract from `PreToolUse`.
