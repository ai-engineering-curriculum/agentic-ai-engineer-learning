# Chapter 3 — Least-Privilege Tool Permissions and Sandboxing

The blast radius of a compromised agent equals the union of every tool it can call. If your support agent holds `read_file`, `run_shell`, and `send_email`, then one successful injection ([Chapter 2](02-prompt-injection.md)) can read secrets, run arbitrary commands, and exfiltrate them. **Least privilege** shrinks that union to exactly what the agent's job requires — nothing more.

```text
   agent role ──▶ permission policy ──▶ allowed tools (+ scoped args)
                        │
                        ▼ a requested tool/arg outside the policy
                     DENY (logged), never executed
```

Two layers do the work: **permissions** decide *whether* a tool may be called with these arguments, and **sandboxing** limits the damage *if* it runs.

## Permissions: scope the tool and its arguments

Scoping the tool name is not enough — most damage is in the arguments. `read_file("/etc/passwd")` and `read_file("./data/report.csv")` are the same tool, wildly different risk.

```python
from dataclasses import dataclass
from pathlib import Path

@dataclass(frozen=True)
class Policy:
    allowed_tools: frozenset[str]
    read_root: Path                  # filesystem confinement
    url_allowlist: frozenset[str]    # exact hosts the agent may fetch

def check_read_file(policy: Policy, path: str) -> bool:
    if "read_file" not in policy.allowed_tools:
        return False
    resolved = (policy.read_root / path).resolve()
    # confine to read_root; .resolve() collapses ../ traversal
    return policy.read_root.resolve() in resolved.parents or resolved == policy.read_root.resolve()
```

Design rules:

- **Allowlist, never blocklist.** Enumerate what's permitted. A blocklist of "bad paths" or "bad hosts" loses to the input you didn't think of.
- **Resolve before you compare.** `../../etc/passwd` looks local until you `.resolve()` it. Canonicalize paths and parse URLs before deciding.
- **Per-role policies.** A `reader` agent gets read-only file access and no network; a `publisher` agent gets the email tool but not the shell. Map roles to policies, not individuals to ad-hoc grants.
- **Deny by default.** An unknown tool or an argument you can't classify is a deny, not an allow.

## Sandboxing: contain what does run

Permissions decide *if*; sandboxing limits *what happens when* — for the genuinely dangerous tools (code execution, shell). Defense in depth, cheapest layer first:

- **Process isolation** — run the tool in a `subprocess` with a restricted working directory, a scrubbed environment (no inherited secrets), no shell (`shell=False`, pass an argv list), and a **timeout** so it can't hang.
- **Resource limits** — cap CPU time and memory (`resource.setrlimit` on POSIX) so a runaway can't exhaust the host.
- **Container/VM isolation** — for untrusted code execution, a disposable container (read-only root, dropped capabilities, no network) is the right boundary. `eval()`/`exec()` in your own process is **not** a sandbox.
- **Network egress control** — a sandbox with open internet can still exfiltrate. Default-deny egress; allowlist the hosts the tool legitimately needs.

```python
import subprocess

def run_sandboxed(argv: list[str], cwd: str, timeout_s: int = 5) -> subprocess.CompletedProcess:
    return subprocess.run(
        argv,                       # argv list, not a shell string
        cwd=cwd,                    # confined working dir
        env={"PATH": "/usr/bin"},  # scrubbed env, no inherited secrets
        capture_output=True, text=True,
        timeout=timeout_s,          # bound runtime
        check=False,
    )
```

## Key takeaways

- Blast radius = the union of an agent's tools; least privilege shrinks it to the job's minimum.
- Scope the **arguments**, not just the tool name — allowlist, resolve/canonicalize, deny by default.
- Map **roles to policies**; don't hand out ad-hoc grants.
- Sandbox dangerous tools with process isolation, resource limits, and egress control; `eval()` is not a sandbox.
