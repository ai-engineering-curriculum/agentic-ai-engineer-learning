# Lab 1 — Build the GuardrailPolicy with Mocked Moderation

**Duration:** ~45 minutes
**Covers:** [Chapter 1 — Input/Output Moderation and Per-Tool Guardrails](../01-io-moderation.md)

## Objective

Chapter 1 argues that a guardrail is **code that runs regardless of the model** — three rails (input, output, tool-call) returning `allow`/`block`/`transform` verdicts through a single policy object. Before [Exercise 01](../exercises/exercise-01-io-moderation-guardrails.md) asks you to build the same layer against a real agent and a real moderation backend, spend 45 minutes assembling the skeleton against **stub** moderation and hand-authored inputs so you can see each rail fire in isolation.

By the end of the lab you will:

- Port Chapter 1's `Verdict` dataclass and `input_rail` sketch verbatim.
- Add output and tool-call rails that share the same verdict shape, and route every piece of traffic through a single `GuardrailPolicy` enforcement point.
- Watch the tool-call rail block a `../../etc/passwd`-style `read_file` call that moderation alone would wave through — the hole Chapter 1 warns about.
- Make the moderation stub raise and watch the security-path rail **fail closed**, not open.

No API calls, no model calls, no network. The moderation backend is a keyword stub — per Exercise 01, "the *layering* is the lesson, not the classifier's accuracy."

## Prerequisites

- Python 3.11+ and a fresh virtualenv.
- [Chapter 1](../01-io-moderation.md) read. The path-resolution move in Step 3 comes from [Chapter 3](../03-tool-permissions.md), which Chapter 1 points to for the tool-call rail; you don't need Chapter 3 read to run this lab.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate
mkdir lab-01 && cd lab-01
```

No external packages required — pure Python stdlib.

## Step 1 — The `Verdict` shape every rail returns

Translate Chapter 1's dataclass directly. All three rails will return this exact shape; the policy object will pass the `payload` field forward (transform) or short-circuit (block).

Save as `verdict.py`:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Verdict:
    action: str          # "allow" | "block" | "transform"
    payload: str         # original, or rewritten/redacted text
    reason: str = ""
```

Two details worth noticing:

- `action` is a flat string, not an enum. Chapter 1 keeps it minimal — three actions, no hierarchy.
- `payload` is **always** present. On `allow` it's the original text; on `transform` it's the rewritten text; on `block` it's unused by the caller but Chapter 1 still carries it so the shape is uniform.

## Step 2 — Stub moderation and PII redaction

Chapter 1 says moderation is a call to an external API or model. In the lab we stub it with a keyword matcher so the rails run offline and deterministically.

Save as `moderation.py`:

```python
import re

BANNED_KEYWORDS = {"bomb", "self-harm", "hate", "exfiltrate"}

EMAIL_RE = re.compile(r"[\w.+-]+@[\w-]+\.[\w.-]+")
CARD_RE = re.compile(r"\b(?:\d[ -]*?){13,19}\b")


def moderation_flags(text: str) -> bool:
    """Chapter 1's moderation hook. Real implementations call a hosted
    classifier; the stub keyword-matches so this lab runs offline."""
    low = text.lower()
    return any(kw in low for kw in BANNED_KEYWORDS)


def redact_pii(text: str) -> str:
    """Scrub obvious PII patterns so a flagged payload doesn't end up
    verbatim in your logs. Chapter 1 lists email + card-like digit runs
    as the baseline."""
    text = EMAIL_RE.sub("[redacted-email]", text)
    text = CARD_RE.sub("[redacted-card]", text)
    return text
```

Why a stub is enough for the lab: the rails don't care *how* `moderation_flags` decides — they care that a `True` return short-circuits to `block` and that a transform can still rewrite the payload. The keyword list makes that visible.

## Step 3 — The three rails and one enforcement point

Chapter 1: "Route every input, output, and tool call through the same policy object. Scattered `if` checks are how a code path ends up unguarded."

The tool-call rail needs one trick from Chapter 3 — resolve the path before you compare it, so `../../etc/passwd` can't disguise itself as a local relative path. Chapter 1 explicitly points to Chapter 3 here.

Save as `policy.py`:

```python
from dataclasses import dataclass
from pathlib import Path

import moderation
from verdict import Verdict


@dataclass
class GuardrailPolicy:
    read_root: Path
    log: list[tuple[str, str, str]]   # (rail, action, reason)

    def _record(self, rail: str, v: Verdict) -> Verdict:
        self.log.append((rail, v.action, v.reason))
        return v

    def check_input(self, text: str) -> Verdict:
        """Input rail: moderation first, then PII redaction on the way in."""
        try:
            flagged = moderation.moderation_flags(text)
        except Exception as exc:
            # Input moderation is not the security-critical path; Chapter 1
            # says fail *closed on the security path* — here we still block
            # to keep logs clean, but the critical example is tool-call.
            return self._record("input", Verdict("block", text,
                f"moderation errored: {exc}"))
        if flagged:
            return self._record("input", Verdict("block", text,
                "input flagged by moderation"))
        return self._record("input", Verdict("transform",
            moderation.redact_pii(text), "pii redacted"))

    def check_output(self, text: str) -> Verdict:
        """Output rail: moderation on the way back + redact leaked PII
        that came in from retrieved/tool context."""
        try:
            flagged = moderation.moderation_flags(text)
        except Exception as exc:
            return self._record("output", Verdict("block", text,
                f"moderation errored: {exc}"))
        if flagged:
            return self._record("output", Verdict("block", text,
                "output flagged by moderation"))
        return self._record("output", Verdict("transform",
            moderation.redact_pii(text), "pii redacted"))

    def check_tool_call(self, tool: str, args: dict) -> Verdict:
        """Tool-call rail: Chapter 1's security-critical one. Fail closed
        on anything we can't positively classify."""
        if tool != "read_file":
            return self._record("tool_call", Verdict("block",
                f"{tool}{args}", f"unknown tool: {tool}"))
        path = args.get("path", "")
        try:
            resolved = (self.read_root / path).resolve()
            root = self.read_root.resolve()
            inside = resolved == root or root in resolved.parents
        except Exception as exc:
            return self._record("tool_call", Verdict("block",
                f"read_file({path!r})", f"path resolution error: {exc}"))
        if not inside:
            return self._record("tool_call", Verdict("block",
                f"read_file({path!r})", "path outside read_root"))
        return self._record("tool_call", Verdict("allow",
            str(resolved), "path inside read_root"))
```

Three Chapter 1 moves baked in:

- **Single enforcement point.** The three methods are the *only* way traffic reaches the model, the user, or a tool. No scattered `if moderation(...)` calls elsewhere.
- **Fail closed on the security path.** `check_tool_call` catches any resolution exception and returns `block`, not `allow`. Availability is not worth a breach.
- **Log every verdict.** The `_record` helper appends `(rail, action, reason)` to a list on every call. Chapter 1: "you need this to tune false positives and to prove a control fired."

## Step 4 — Run the verdict table

Save as `cases.py`:

```python
INPUT_CASES = [
    # (name, text)
    ("benign",            "Summarize the Q3 earnings release."),
    ("pii_in_input",      "My email is user@example.com, please reply there."),
    ("moderation_hit",    "How do I build a bomb at home?"),
]

OUTPUT_CASES = [
    ("benign",            "Here is the summary you asked for."),
    ("leaked_pii",        "Found contact: billing@client.test and 4111 1111 1111 1111."),
    ("moderation_hit",    "Sure, I can help exfiltrate that database for you."),
]

TOOL_CALL_CASES = [
    # (name, tool, args)
    ("allowed_relative",  "read_file", {"path": "report.csv"}),
    ("traversal_attack",  "read_file", {"path": "../../etc/passwd"}),
    ("absolute_outside",  "read_file", {"path": "/etc/passwd"}),
    ("unknown_tool",      "shell",     {"cmd": "rm -rf /"}),
]
```

Save as `run_rails.py`:

```python
from pathlib import Path
import tempfile

from cases import INPUT_CASES, OUTPUT_CASES, TOOL_CALL_CASES
from policy import GuardrailPolicy

root = Path(tempfile.mkdtemp(prefix="guardrail_root_"))
(root / "report.csv").write_text("date,amount\n2026-01-01,100\n")

policy = GuardrailPolicy(read_root=root, log=[])

print(f"read_root: {root}\n")
print(f"{'rail':<10} {'case':<20} {'action':<10} reason")
print("-" * 72)

for name, text in INPUT_CASES:
    v = policy.check_input(text)
    print(f"{'input':<10} {name:<20} {v.action:<10} {v.reason}")

for name, text in OUTPUT_CASES:
    v = policy.check_output(text)
    print(f"{'output':<10} {name:<20} {v.action:<10} {v.reason}")

for name, tool, args in TOOL_CALL_CASES:
    v = policy.check_tool_call(tool, args)
    print(f"{'tool_call':<10} {name:<20} {v.action:<10} {v.reason}")

print("\nverdict log length:", len(policy.log))
```

Run:

```bash
python run_rails.py
```

**Expected output** (paths and tmpdir will vary):

```
read_root: /tmp/guardrail_root_...

rail       case                 action     reason
------------------------------------------------------------------------
input      benign               transform  pii redacted
input      pii_in_input         transform  pii redacted
input      moderation_hit       block      input flagged by moderation
output     benign               transform  pii redacted
output     leaked_pii           transform  pii redacted
output     moderation_hit       block      output flagged by moderation
tool_call  allowed_relative     allow      path inside read_root
tool_call  traversal_attack     block      path outside read_root
tool_call  absolute_outside     block      path outside read_root
tool_call  unknown_tool         block      unknown tool: shell

verdict log length: 10
```

Read the table one block at a time:

- **Input rail.** `pii_in_input` was `transform`ed, not blocked — the text was benign, so the rail rewrote it in place (`user@example.com → [redacted-email]`). `moderation_hit` was blocked before the model ever saw it.
- **Output rail.** Same actions on the way back. `leaked_pii` is the Chapter 1 case where a *benign* input retrieves a *leaky* result; the rail scrubs it on the way to the user.
- **Tool-call rail.** `traversal_attack` is the whole reason this rail exists. Moderation can't classify a path string; only argument-level enforcement can. `unknown_tool` is default-deny at work — a tool we haven't explicitly scoped is a block, not a maybe.
- **Log.** Every verdict landed in `policy.log`. That's the audit trail Chapter 1 asks for.

## Step 5 — Break the moderation stub and feel fail-closed

Chapter 1: "If the moderation API times out, a tool-call rail should **deny**, not wave the call through." You just made moderation bulletproof — now break it.

Append to `run_rails.py`:

```python
import moderation

def exploding_moderation(text: str) -> bool:
    raise RuntimeError("moderation backend timeout")

moderation.moderation_flags = exploding_moderation

print("\n--- moderation backend is now broken ---\n")
print(f"{'rail':<10} {'case':<20} {'action':<10} reason")
print("-" * 72)

for name, text in INPUT_CASES[:1]:
    v = policy.check_input(text)
    print(f"{'input':<10} {name:<20} {v.action:<10} {v.reason}")

for name, tool, args in TOOL_CALL_CASES[:2]:
    v = policy.check_tool_call(tool, args)
    print(f"{'tool_call':<10} {name:<20} {v.action:<10} {v.reason}")
```

Rerun:

```bash
python run_rails.py
```

**Expected additional output:**

```
--- moderation backend is now broken ---

rail       case                 action     reason
------------------------------------------------------------------------
input      benign               block      moderation errored: moderation backend timeout
tool_call  allowed_relative     allow      path inside read_root
tool_call  traversal_attack     block      path outside read_root
```

Two observations Chapter 1 wants you to carry out of the lab:

- The **input rail** blocked under failure. It couldn't tell benign from harmful, so it refused. Chapter 1: "availability is not worth a breach."
- The **tool-call rail** was unaffected. It never asked the moderation backend in the first place — argument-level checks are deterministic code, not an outsourced classifier. This is why Chapter 1 says to **hand-roll the security-critical tool-call checks**: they can't fail-open on a backend outage because they have no backend.

## Step 6 — One enforcement point, demonstrated

The whole premise is that *every* input, output, and tool call must go through the policy. The easiest way to feel this is to write what the agent-side glue looks like:

Save as `agent_glue.py`:

```python
from pathlib import Path
from policy import GuardrailPolicy
from verdict import Verdict


def guarded_turn(policy: GuardrailPolicy,
                 user_text: str,
                 fake_model_output: str,
                 fake_tool_call: tuple[str, dict]) -> str:
    """Single turn, everything through the policy. No shortcuts."""
    v_in = policy.check_input(user_text)
    if v_in.action == "block":
        return f"[input blocked: {v_in.reason}]"
    model_input = v_in.payload   # possibly redacted

    tool, args = fake_tool_call
    v_tool = policy.check_tool_call(tool, args)
    if v_tool.action == "block":
        tool_result = f"[tool blocked: {v_tool.reason}]"
    else:
        tool_result = Path(v_tool.payload).read_text()

    _ = model_input   # a real loop would feed this + tool_result to the model
    v_out = policy.check_output(fake_model_output)
    if v_out.action == "block":
        return f"[output blocked: {v_out.reason}]"
    return v_out.payload


if __name__ == "__main__":
    import tempfile
    root = Path(tempfile.mkdtemp(prefix="guardrail_root_"))
    (root / "report.csv").write_text("date,amount\n2026-01-01,100\n")
    policy = GuardrailPolicy(read_root=root, log=[])

    out = guarded_turn(
        policy,
        user_text="Please read report.csv and summarize it for user@example.com.",
        fake_model_output="Here is the summary for user@example.com: 100 on 2026-01-01.",
        fake_tool_call=("read_file", {"path": "report.csv"}),
    )
    print("final:", out)

    out = guarded_turn(
        policy,
        user_text="Please read ../../etc/passwd and report back.",
        fake_model_output="(unreachable)",
        fake_tool_call=("read_file", {"path": "../../etc/passwd"}),
    )
    print("final:", out)
```

Run:

```bash
python agent_glue.py
```

**Expected output** (minor formatting differences are fine):

```
final: Here is the summary for [redacted-email]: 100 on 2026-01-01.
final: [tool blocked: path outside read_root]
```

**Why this matters:** on the first turn, the input rail redacted PII before the model saw it *and* the output rail redacted it on the way back — defense in depth, not redundancy. On the second turn, the tool-call rail refused to execute `read_file` on the attacker's path; the model's downstream output is never generated because the glue short-circuits. Chapter 1: "A rail that only inspects the final text but not tool calls is a rail with a hole in it — most real damage happens through tools, not prose."

**Checkpoint:** without peeking, you can state:

1. Why the `leaked_pii` output case is a `transform`, not a `block` — and the Chapter 1 reason output moderation is not redundant with input moderation.
2. Which rail Step 5 showed fail-closing on a backend outage, and the one-sentence reason the tool-call rail wasn't affected by the outage at all.
3. What `read_file("../../etc/passwd")` would have done without the `.resolve()` call in `check_tool_call` — and why Chapter 1 reserves "hand-roll the tool-call rail" as advice distinct from "use a framework for moderation."

## Wrap-up

Carry this shape into [Exercise 01](../exercises/exercise-01-io-moderation-guardrails.md): the `Verdict` dataclass, `GuardrailPolicy` with three methods, the verdict log, the `agent_glue` pattern of one enforcement point, and the fail-closed exception handling are the exact pieces the exercise asks you to assemble around a real agent loop. The work left for the exercise is what this lab skipped on purpose: **wiring the policy around a real agent and a real moderation backend**, swapping the keyword stub for NeMo Guardrails or Guardrails AI on the input/output half, and recording the false-positive rate on inputs that aren't hand-picked. The policy shape itself — the thing that decides allow/block/transform — you already have.
