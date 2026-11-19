# Lab 3 — Gate High-Risk Tool Calls with a Scripted Approver

**Duration:** ~45 minutes
**Covers:** [Chapter 4 — Human-Approval Checkpoints for High-Risk Actions](../04-human-approval.md)

## Objective

Chapter 4's thesis: for actions you can't take back, the right control is a **human in the loop**, pausing the agent before the irreversible step runs. Before [Exercise 04](../exercises/exercise-04-human-approval-checkpoints.md) asks you to wire the checkpoint against a real agent with a CLI prompt, spend 45 minutes building the checkpoint mechanics against a **scripted approver** — a Python stand-in that returns pre-set decisions — so every detail (classification, pause, timeout, replan feedback, audit log) is deterministic and testable.

By the end of the lab you will:

- Port Chapter 4's `HIGH_RISK_TOOLS` set and `needs_approval(tool, args)` classifier verbatim, including an argument-level threshold.
- Build a `request_approval` that **shows the full concrete action** before capturing a decision, and watch what happens when it doesn't.
- Simulate a timeout and feel fail-closed: no decision → **denied**, not approved.
- Feed a `denied` outcome back through a scripted agent loop and watch it **replan** instead of re-requesting the same call.
- Persist an audit trail of every decision.

No API calls, no `input()`, no network. The scripted approver swaps in for a human so the Chapter 4 mechanics are what you're reading.

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 4](../04-human-approval.md) read.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-03 && cd lab-03
```

No external packages required — pure Python stdlib.

## Step 1 — Classify risk by tool **and** arguments

Chapter 4: "Make the gate a property of the *tool plus its arguments*, not a vibe." Port the classifier directly.

Save as `classify.py`:

```python
HIGH_RISK_TOOLS = {"send_email", "transfer_funds", "delete_file", "deploy", "publish"}

INTERNAL_DOMAIN = "corp.test"


def needs_approval(tool: str, args: dict) -> bool:
    """Chapter 4's gate. Three shapes of rule:

    - always-gate tools (unconditional irreversibility)
    - threshold gates (reversible until a dollar amount)
    - recipient-sensitive gates (send_email outside the org)
    """
    if tool in HIGH_RISK_TOOLS:
        # Narrow one case: send_email inside the org auto-runs.
        if tool == "send_email":
            to = args.get("to", "")
            if to.endswith("@" + INTERNAL_DOMAIN):
                return False
        return True
    if tool == "spend" and args.get("amount", 0) > 100:
        return True
    return False
```

Three Chapter-4-aligned choices:

- `HIGH_RISK_TOOLS` is a **set of names**. A tool you haven't classified does not default to high-risk here — and Chapter 4 is explicit that gating everything trains humans to rubber-stamp. The classifier earns its specificity by enumerating what's irreversible.
- `send_email` is *not* unconditional. It gates on the recipient's domain — exactly Chapter 4's "`send_email` gates when the recipient is outside your domain." Approval fatigue is a real failure mode; don't pause for every internal cc.
- `spend` is a **threshold gate**: the same tool is auto-run under `$100` and gated over it. The tool name alone can't tell you that; the argument has to.

## Step 2 — The approval shape and a scripted approver

Chapter 4: "Capture a decision, not a guess. Record `approved`/`denied`, who decided, when, and any edited arguments. This is your audit trail."

Save as `approval.py`:

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone


@dataclass(frozen=True)
class Decision:
    approved: bool
    decided_by: str
    decided_at: str
    reason: str = ""


@dataclass
class ApprovalRequest:
    tool: str
    args: dict
    why: str                         # the agent's stated reason
    received_at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())


class ScriptedApprover:
    """Test-time replacement for a human. Looks up a decision by a (tool, key)
    pair. A missing entry means 'no response within the window' — the caller
    must treat that as denial, per Chapter 4's fail-closed rule."""

    def __init__(self, name: str, responses: dict[tuple[str, str], Decision]):
        self.name = name
        self.responses = responses

    def decide(self, req: ApprovalRequest) -> Decision | None:
        key = (req.tool, _key_of(req.args))
        return self.responses.get(key)


def _key_of(args: dict) -> str:
    """Stable key for looking up a scripted decision. Chapter 4 wants the
    *concrete call* approved, so we key on the exact args."""
    return "|".join(f"{k}={args[k]}" for k in sorted(args))
```

The approver is scripted so the lab is reproducible. In Exercise 04 you'll swap it for a CLI `input()` prompt; the surrounding mechanics don't change.

## Step 3 — The checkpoint: pause, show the full action, decide

Chapter 4's checkpoint has four pieces: (a) **show** the exact action, not a vague summary; (b) **capture** the decision; (c) on timeout, **fail closed**; (d) emit an outcome the agent can route on.

Save as `checkpoint.py`:

```python
from datetime import datetime, timezone

from approval import ApprovalRequest, Decision, ScriptedApprover


def _now() -> str:
    return datetime.now(timezone.utc).isoformat()


def present_action(req: ApprovalRequest) -> str:
    """Chapter 4: show tool, exact arguments, agent's stated reason.
    'Approve action?' with no details trains people to click yes."""
    lines = [
        f"PROPOSED ACTION: {req.tool}",
        "arguments:",
    ]
    for k in sorted(req.args):
        lines.append(f"  - {k} = {req.args[k]!r}")
    lines.append(f"reason: {req.why}")
    return "\n".join(lines)


def request_approval(req: ApprovalRequest,
                     approver: ScriptedApprover) -> Decision:
    """Pause, show the action, capture a decision, fail closed on timeout."""
    print("---- checkpoint ----")
    print(present_action(req))
    result = approver.decide(req)
    if result is None:
        # Chapter 4: "No response within the window → treat as denied."
        denial = Decision(approved=False, decided_by="runtime",
                          decided_at=_now(), reason="timeout: no decision")
        print(f"decision: DENIED (fail-closed, {denial.reason})")
        return denial
    label = "APPROVED" if result.approved else "DENIED"
    print(f"decision: {label} by {result.decided_by} — {result.reason}")
    return result
```

Two Chapter 4 points visible in the code:

- The printed block is the **concrete** call, not an abstract plan. Chapter 4's pitfall list: "Approve the *concrete tool call*, not a high-level plan. The plan can say 'email the customer' while the argument says 'email attacker@evil.com.'"
- The `None` branch is the **fail-closed** case. The default is not "approve" or "ask again" — it is `denied`, with reason recorded. An approval that defaults to yes is a backdoor.

## Step 4 — The mini agent loop

Wire the classifier, the checkpoint, and a simple tool-executor stub into one loop. The "agent" is a scripted planner that picks a next step given the user ask and prior results — enough to show that **deny → replan** actually loops back.

Save as `agent.py`:

```python
from dataclasses import dataclass, field
from datetime import datetime, timezone

from approval import ApprovalRequest, Decision, ScriptedApprover
from checkpoint import request_approval
from classify import needs_approval


@dataclass
class ToolCall:
    tool: str
    args: dict
    why: str


@dataclass
class AuditEntry:
    tool: str
    args: dict
    decision: Decision | None
    outcome: str
    at: str = field(default_factory=lambda: datetime.now(timezone.utc).isoformat())


def execute_tool(call: ToolCall) -> str:
    """Stub executor. Real tools would do the work; the lab just records."""
    return f"[ran {call.tool}({call.args})]"


def step(call: ToolCall,
         approver: ScriptedApprover,
         audit: list[AuditEntry]) -> str:
    """One turn: classify, gate if needed, execute or report denial."""
    if not needs_approval(call.tool, call.args):
        outcome = execute_tool(call)
        audit.append(AuditEntry(tool=call.tool, args=call.args,
                                decision=None, outcome=outcome))
        return outcome

    req = ApprovalRequest(tool=call.tool, args=call.args, why=call.why)
    decision = request_approval(req, approver)
    if not decision.approved:
        # Chapter 4: "On deny, return a tool result the loop can replan on."
        outcome = f"action denied by human: {decision.reason}"
    else:
        outcome = execute_tool(call)
    audit.append(AuditEntry(tool=call.tool, args=call.args,
                            decision=decision, outcome=outcome))
    return outcome


def scripted_plan(user_ask: str, prior: list[str]) -> ToolCall | None:
    """Tiny planner. First attempt: a borderline send_email outside the org.
    If that was denied, fall back to a safe internal notification.
    Once something has succeeded (or we've run out of safe moves), stop."""
    if "notify billing" not in user_ask.lower():
        return None
    if any(p.startswith("[ran") for p in prior):
        return None   # the task is done
    denies = sum("denied" in p for p in prior)
    if denies == 0:
        return ToolCall(
            tool="send_email",
            args={"to": "billing@vendor.test",
                  "subject": "Invoice bump",
                  "body": "FYI bumping invoice #42 by $200."},
            why="user asked to notify billing of the bump",
        )
    if denies == 1:
        return ToolCall(
            tool="send_email",
            args={"to": "ap@corp.test",
                  "subject": "Invoice bump — please notify billing",
                  "body": "Bumping invoice #42 by $200; please forward."},
            why="external recipient denied; routing through internal AP",
        )
    return None   # out of safe moves; stop looping
```

Three Chapter 4 takeaways baked into `scripted_plan`:

- It **counts prior denies** instead of blindly retrying. Chapter 4: "On deny, return a tool result like 'action denied by human' so the loop can replan instead of retrying the same call forever."
- The replan is **not the same call with a different plea** — it's a genuinely different action (internal recipient instead of external). A loop that retries the identical call is what the deny-feedback is supposed to prevent.
- After the second deny, the planner **stops** and returns `None`. Infinite replan is just infinite approval requests; the floor is "stop and ask a human the one question the agent can't answer."

## Step 5 — Run the gate: who passes, who pauses

Save as `run_gate.py`:

```python
from agent import ToolCall, step
from approval import Decision, ScriptedApprover
from classify import needs_approval


# A mix of calls: low-risk, inside-domain email (auto), outside-domain email,
# spend under + over the threshold, delete_file.
CALLS = [
    ToolCall(tool="search",       args={"q": "Q3 revenue"},                    why="user asked for a number"),
    ToolCall(tool="send_email",   args={"to": "ops@corp.test", "body": "ok"},  why="internal note"),
    ToolCall(tool="send_email",   args={"to": "steve@vendor.test", "body": "hi"}, why="status update"),
    ToolCall(tool="spend",        args={"amount": 42},                         why="under threshold"),
    ToolCall(tool="spend",        args={"amount": 420},                        why="over threshold"),
    ToolCall(tool="delete_file",  args={"path": "logs/2024/"},                 why="rotate old logs"),
]


print("needs_approval classifier:")
for c in CALLS:
    print(f"  {c.tool:<14} {c.args}  -> {needs_approval(c.tool, c.args)}")
```

Run:

```bash
python run_gate.py
```

**Expected output:**

```
needs_approval classifier:
  search         {'q': 'Q3 revenue'}  -> False
  send_email     {'to': 'ops@corp.test', 'body': 'ok'}  -> False
  send_email     {'to': 'steve@vendor.test', 'body': 'hi'}  -> True
  spend          {'amount': 42}  -> False
  spend          {'amount': 420}  -> True
  delete_file    {'path': 'logs/2024/'}  -> True
```

The two things to read off:

- **Argument-level discrimination.** Two `send_email` calls, same tool, different fate — because of the recipient's domain. Same for `spend` across the threshold. Chapter 4: "Gate by the tool **and its arguments**."
- **Reflex vs. real control.** `search` doesn't pause. Neither does an internal cc. If every call paused, people would click yes without reading. Chapter 4's "approval fatigue" risk is averted by *not* gating everything — the classifier earns the pause.

## Step 6 — Approve, deny, time out — and see fail-closed

Now wire the approver. Give it one `approved` response and one `denied` response, and leave one case **unanswered** — the scripted stand-in for timeout.

Append to `run_gate.py`:

```python
from datetime import datetime, timezone

from agent import AuditEntry, step
from approval import Decision, ScriptedApprover, _key_of


def _now() -> str:
    return datetime.now(timezone.utc).isoformat()


# For each (tool, key_of(args)) we want to respond to, pre-record a Decision.
# A missing entry simulates no response → timeout → fail-closed denial.
responses = {
    ("send_email", _key_of({"to": "steve@vendor.test", "body": "hi"})):
        Decision(approved=True, decided_by="alice", decided_at=_now(),
                 reason="recipient known"),
    ("spend", _key_of({"amount": 420})):
        Decision(approved=False, decided_by="alice", decided_at=_now(),
                 reason="over policy — needs CFO sign-off"),
    # delete_file deliberately missing → fail-closed on timeout.
}

approver = ScriptedApprover(name="alice", responses=responses)
audit: list[AuditEntry] = []

print("\n--- running the calls ---\n")
for call in CALLS:
    outcome = step(call, approver, audit)
    print(f"-> {outcome}\n")

print("audit length:", len(audit))
```

Run:

```bash
python run_gate.py
```

**Expected output (abbreviated to the interesting lines):**

```
--- running the calls ---

-> [ran search({'q': 'Q3 revenue'})]

-> [ran send_email({'to': 'ops@corp.test', 'body': 'ok'})]

---- checkpoint ----
PROPOSED ACTION: send_email
arguments:
  - body = 'hi'
  - to = 'steve@vendor.test'
reason: status update
decision: APPROVED by alice — recipient known
-> [ran send_email({'to': 'steve@vendor.test', 'body': 'hi'})]

-> [ran spend({'amount': 42})]

---- checkpoint ----
PROPOSED ACTION: spend
arguments:
  - amount = 420
reason: over threshold
decision: DENIED by alice — over policy — needs CFO sign-off
-> action denied by human: over policy — needs CFO sign-off

---- checkpoint ----
PROPOSED ACTION: delete_file
arguments:
  - path = 'logs/2024/'
reason: rotate old logs
decision: DENIED (fail-closed, timeout: no decision)
-> action denied by human: timeout: no decision

audit length: 6
```

Three Chapter 4 moments to notice:

- Three **different** outcomes for high-risk calls: approved and executed, denied and reported, timed-out and fail-closed. The agent never sees the difference between "no" and "no answer" — both come back as "denied."
- The checkpoint printed the **full concrete action** before any decision was made. An approver who sees only "approve the plan?" could have missed the outside-domain recipient or the `420` amount. The arguments are the attack surface.
- Every outcome — approved, denied, timed-out — landed in `audit`. Chapter 4: "This is your audit trail."

## Step 7 — Deny → replan: the agent loops back, doesn't bang on the door

The per-call demo in Step 6 was linear. Now let the scripted planner react to a deny and try an alternative, so you see Chapter 4's "feed the outcome back" rule do its job.

Save as `run_replan.py`:

```python
from datetime import datetime, timezone

from agent import AuditEntry, scripted_plan, step
from approval import Decision, ScriptedApprover, _key_of


def _now() -> str:
    return datetime.now(timezone.utc).isoformat()


# Approver: deny the external email, approve the internal fallback.
responses = {
    ("send_email", _key_of({"body": "FYI bumping invoice #42 by $200.",
                            "subject": "Invoice bump",
                            "to": "billing@vendor.test"})):
        Decision(approved=False, decided_by="alice", decided_at=_now(),
                 reason="external recipient not approved for this flow"),
}

approver = ScriptedApprover(name="alice", responses=responses)
audit: list[AuditEntry] = []
prior: list[str] = []
user_ask = "notify billing about the bump"

for turn in range(4):
    call = scripted_plan(user_ask, prior)
    if call is None:
        print("\nagent has nothing more to propose; loop ends.")
        break
    print(f"\n--- turn {turn}: agent proposes {call.tool} ---")
    outcome = step(call, approver, audit)
    prior.append(outcome)
    print(f"-> {outcome}")
```

Run:

```bash
python run_replan.py
```

**Expected output:**

```

--- turn 0: agent proposes send_email ---
---- checkpoint ----
PROPOSED ACTION: send_email
arguments:
  - body = 'FYI bumping invoice #42 by $200.'
  - subject = 'Invoice bump'
  - to = 'billing@vendor.test'
reason: user asked to notify billing of the bump
decision: DENIED by alice — external recipient not approved for this flow
-> action denied by human: external recipient not approved for this flow

--- turn 1: agent proposes send_email ---
-> [ran send_email({'to': 'ap@corp.test', 'subject': 'Invoice bump — please notify billing', 'body': 'Bumping invoice #42 by $200; please forward.'})]

agent has nothing more to propose; loop ends.
```

Read the trace:

- Turn 0: external `send_email` was gated, **denied**, and returned to the planner as `"action denied by human: ..."`.
- Turn 1: the planner **did not retry the same call**. It picked a semantically different action — an internal address — that `needs_approval` auto-approves (recipient inside `corp.test`). The gate did its job; the agent made progress.
- Turn 2 (no call): with the task completed, the planner returns `None` and the loop ends. A buggy planner that re-requested the identical denied call would just keep asking the approver the same question forever. Chapter 4's "replan instead of retry" is precisely what stops a denied agent from becoming a denial-of-service against your approvers.

## Step 8 — The audit log is the receipt

Append to `run_replan.py`:

```python
print("\n--- audit trail ---")
for entry in audit:
    decision = entry.decision
    if decision is None:
        line = f"{entry.at} AUTO {entry.tool}({entry.args})"
    else:
        label = "APPROVED" if decision.approved else "DENIED"
        line = (f"{entry.at} {label} by {decision.decided_by} "
                f"{entry.tool}({entry.args}) — {decision.reason}")
    print(line)
```

Rerun. **Expected output** (timestamps and dict order from your Python version may vary):

```
--- audit trail ---
2026-11-19T...+00:00 DENIED by alice send_email({'to': 'billing@vendor.test', ...}) — external recipient not approved for this flow
2026-11-19T...+00:00 AUTO send_email({'to': 'ap@corp.test', ...})
```

Two Chapter 4 points to notice:

- The audit contains **both** the denied attempt and the auto-approved fallback. You cannot replay the agent's turn from the approved calls alone — you need the denials to understand *why* the chosen path happened.
- The AUTO entry has no `decision`. The audit still records it, because the trail Chapter 4 names is every *decision point*, not just every human interaction — the gate's "no approval needed" is itself a decision.

**Checkpoint:** without peeking, you can state:

1. The two cases where the **same tool** (`send_email`, or `spend`) produced different gate outcomes in Step 5, and the Chapter 4 phrasing for why gating is a property of the tool *plus* the arguments.
2. Why a missing approval in Step 6 fell through to `denied`, not `retry` or `approved`, and the one-sentence Chapter 4 rule that pins that choice.
3. The difference between what the planner did on turn 1 of Step 7 vs. what a buggy "retry the same call" loop would have done — and the Chapter 4 rule that prescribes the first behavior.

## Wrap-up

Carry this shape into [Exercise 04](../exercises/exercise-04-human-approval-checkpoints.md): the `HIGH_RISK_TOOLS` + `needs_approval` classifier, the `ApprovalRequest`/`Decision` pair, the `request_approval` function with fail-closed timeout, the "return a `denied` result so the loop can replan" pattern, and the audit list are exactly the pieces the exercise asks you to build around a real agent. The work left for the exercise is what this lab skipped on purpose: **wiring the checkpoint against a real agent loop and a CLI (or richer UI) approver**, tuning the classifier against your scenario's threshold, and running the stretch goal of letting the approver edit arguments before approving. The gate mechanics — the piece that decides *when* to pause, *what* to show, and *how* to react — you already have.
