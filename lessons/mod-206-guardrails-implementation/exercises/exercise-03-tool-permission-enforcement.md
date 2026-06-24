# exercise-03: Tool-Permission Enforcement

**Estimated effort:** 2 hours

## Objective

Enforce least-privilege tool permissions for an agent: map roles to policies, scope tool *arguments* (not just tool names), and deny every out-of-policy call by default. Then sandbox the one genuinely dangerous tool so that even an allowed call can't escape its blast radius. By the end you'll have a permission layer where a `reader` role physically cannot reach the actions a `publisher` role can.

## Background

This exercise covers material from:

- [Chapter 3 — Least-Privilege Tool Permissions and Sandboxing](../03-tool-permissions.md)

Use the agent loop from [mod-201](../../mod-201-agent-fundamentals/README.md). You'll define two roles and three tools and show the policy layer enforcing the boundary regardless of what the model asks for.

## Prerequisites

- An agent loop with tool calling (from mod-201).
- Three tools: `read_file(path)`, `fetch_url(url)`, and `run_code(source)`.
- POSIX shell for the sandboxing task (`subprocess`, optionally `resource`); a container runtime is a stretch goal.

## Tasks

### 1. Role-to-policy map

- Define a `Policy` with `allowed_tools`, a filesystem `read_root`, and a `url_allowlist`.
- Define two roles: `reader` (read files under `read_root`, no network, no code) and `publisher` (read + `fetch_url` to allowlisted hosts, still no `run_code`). Map role → policy.

### 2. Argument-level enforcement

- `read_file`: resolve the path and **deny** anything outside `read_root` — block `../` traversal and absolute paths. Allowlist, not blocklist.
- `fetch_url`: parse the URL and **deny** any host not on the allowlist. Don't substring-match; compare the parsed host.
- Default-deny: an unknown tool or an argument you can't classify is a deny.

### 3. Prove the boundary

- Have the `reader` agent attempt a `fetch_url` and a `run_code` call. Both must be denied *by the policy layer*, not by the model declining.
- Have the `publisher` attempt `fetch_url("http://evil.example")` (off-allowlist) and confirm denial.

### 4. Sandbox the dangerous tool

- For `run_code` (only ever available to a future `executor` role), run it in a `subprocess` with: an argv list (no `shell=True`), a confined `cwd`, a scrubbed `env` (no inherited secrets), and a **timeout**. Add a CPU/memory `setrlimit` if on POSIX.
- Show that a `while True:` payload is killed by the timeout and a secret in the parent env is not visible to the child.

### 5. Log denials

- Log every denied call with role, tool, args, and reason. This is the evidence the control fired.

## Starter guidance

```python
from dataclasses import dataclass
from pathlib import Path
from urllib.parse import urlparse
import subprocess

@dataclass(frozen=True)
class Policy:
    allowed_tools: frozenset[str]
    read_root: Path
    url_allowlist: frozenset[str]

def allow_read(policy: Policy, path: str) -> bool:
    if "read_file" not in policy.allowed_tools:
        return False
    target = (policy.read_root / path).resolve()
    root = policy.read_root.resolve()
    return target == root or root in target.parents

def allow_fetch(policy: Policy, url: str) -> bool:
    if "fetch_url" not in policy.allowed_tools:
        return False
    return urlparse(url).hostname in policy.url_allowlist

def run_sandboxed(argv: list[str], cwd: str, timeout_s: int = 5) -> subprocess.CompletedProcess:
    return subprocess.run(argv, cwd=cwd, env={"PATH": "/usr/bin"},
                          capture_output=True, text=True, timeout=timeout_s, check=False)
```

You do **not** need moderation rails (exercise-01) or approval prompts (exercise-04) here.

## Acceptance criteria

You can demonstrate that:

- Two roles map to two policies; the `reader` is strictly a subset of the `publisher`.
- A path-traversal `read_file` and an off-allowlist `fetch_url` are **denied** by the policy layer.
- The `reader` cannot reach `fetch_url` or `run_code` regardless of what the model requests.
- `run_code` runs sandboxed: a runaway is killed by the timeout, and parent-env secrets are not visible to the child.
- Every denial is logged with role, tool, args, and reason.

## Reflection

In `NOTES.md`:

1. Where did argument-level scoping catch something that tool-name scoping would have missed?
2. Why is `eval()`/`exec()` in-process not a sandbox? What did `subprocess` give you that it can't?
3. If you added a fourth tool tomorrow, what's the smallest policy change that keeps default-deny intact?

## Stretch goals

- Run `run_code` in a disposable container (read-only root, dropped capabilities, no network) instead of a subprocess.
- Add network egress control so even an allowed tool can only reach allowlisted hosts.
- Generate the role→policy map from a YAML config and validate it on startup.
