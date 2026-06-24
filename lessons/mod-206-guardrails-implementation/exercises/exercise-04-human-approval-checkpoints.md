# exercise-04: Human-Approval Checkpoints

**Estimated effort:** 2 hours

## Objective

Add human-approval checkpoints to an agent so that high-risk, irreversible actions pause for a person to approve before they run. You'll classify risk by tool and arguments, surface the full concrete action to the approver, capture an auditable decision, fail closed on timeout, and feed the outcome back so the agent replans on denial. By the end you'll have the backstop that holds even when every other guardrail is bypassed.

## Background

This exercise covers material from:

- [Chapter 4 — Human-Approval Checkpoints for High-Risk Actions](../04-human-approval.md)

Use the agent loop from [mod-201](../../mod-201-agent-fundamentals/README.md). The approver can be a CLI `input()` prompt; the point is the *mechanics*, not the UI.

## Prerequisites

- An agent loop with tool calling (from mod-201).
- A mix of tools: low-risk (`search`, `read_file`) and high-risk (`send_email`, `delete_file`, `spend`).
- No external API strictly required if you stub tool execution.

## Tasks

### 1. Risk classifier

- Write `needs_approval(tool, args)` keyed on the tool **and** its arguments: `delete_file` always gates; `spend` gates above a threshold; `send_email` gates when the recipient is outside your domain. Low-risk tools auto-run.

### 2. The checkpoint

- Before a high-risk call executes, pause and present the **full concrete action**: exact tool, exact arguments, and the agent's stated reason. A bare "Approve? [y/n]" is not enough — show the recipient, the amount, the path.
- Capture a decision record: `decision`, `decided_by`, `decided_at`, and any edited arguments.

### 3. Fail closed

- If no decision arrives within a timeout window, treat the action as **denied**. Show that a timed-out approval does not execute.

### 4. Close the loop

- On deny, return a tool result like `"action denied by human: <reason>"` so the agent can replan instead of re-requesting the identical call. Show the agent does something sensible (asks the user, picks a safe alternative) rather than looping.

### 5. Audit trail

- Persist every checkpoint decision (approved and denied) to a log. This pairs with the observability work in [mod-205](../../mod-205-evaluation-observability/README.md).

## Starter guidance

```python
from dataclasses import dataclass
from datetime import datetime, timezone

HIGH_RISK_TOOLS = {"send_email", "delete_file", "deploy", "publish"}

@dataclass(frozen=True)
class Decision:
    approved: bool
    decided_by: str
    decided_at: str
    reason: str = ""

def needs_approval(tool: str, args: dict) -> bool:
    if tool in HIGH_RISK_TOOLS:
        return True
    if tool == "spend" and args.get("amount", 0) > 100:
        return True
    return False

def request_approval(tool: str, args: dict, why: str, timeout_s: int = 60) -> Decision:
    raise NotImplementedError    # show full action; capture y/n; fail closed on timeout

def now() -> str:
    return datetime.now(timezone.utc).isoformat()
```

You do **not** need moderation rails (exercise-01) or sandboxing (exercise-03) here, though they compose.

## Acceptance criteria

You can demonstrate that:

- Low-risk tools auto-run; high-risk tools and over-threshold spends pause for approval.
- The checkpoint shows the **full concrete action** (recipient, amount, path), not a vague summary.
- An approved action executes; a denied one does not; a timed-out one fails **closed** (denied).
- On denial, the agent replans instead of re-requesting the identical call.
- Every decision is persisted with who, when, and the outcome.

## Reflection

In `NOTES.md`:

1. Give a case where approving the *plan* would have been unsafe but approving the *concrete call* caught the problem (hint: a hijacked argument).
2. How did you tune the classifier to avoid approval fatigue without leaving an irreversible action ungated?
3. What's the right default on timeout for *your* scenario, and why is "approve" never it?

## Stretch goals

- Let the approver **edit** arguments before approving (e.g., correct a recipient) and run the edited call.
- Add a per-action expiry so a stale approval can't be replayed later.
- Combine with exercise-02: route an injection-flagged `send_email` to a checkpoint instead of a hard deny, and observe how often the human catches it.
