# exercise-01: Input/Output Moderation Guardrails

**Estimated effort:** 3 hours

## Objective

Build a guardrail layer that moderates input, output, and tool calls around an agent. You'll implement it once with a framework (NeMo Guardrails or Guardrails AI) and once by hand, route every input/output/tool call through a single policy object, and prove that blocked traffic never reaches the model, the user, or a tool. By the end you'll know exactly where a rail belongs and why the tool-call rail is the one that matters most.

## Background

This exercise covers material from:

- [Chapter 1 — Input/Output Moderation and Per-Tool Guardrails](../01-io-moderation.md)

Use any model provider and the agent loop from [mod-201](../../mod-201-agent-fundamentals/README.md). You may stub the moderation backend with a small keyword/regex classifier if you don't want to spend on a moderation API — the *layering* is the lesson, not the classifier's accuracy.

## Prerequisites

- An agent loop with tool calling (from mod-201).
- One tool with a security-relevant argument, e.g. `read_file(path)`.
- `pip install nemoguardrails` **or** `pip install guardrails-ai` for the framework half.
- API key in an environment variable; small spend cap.

## Tasks

### 1. A single policy object

- Define `GuardrailPolicy` with three methods: `check_input(text)`, `check_output(text)`, and `check_tool_call(tool, args)`, each returning a `Verdict(action, payload, reason)` where `action` is `allow`, `block`, or `transform`.
- Wire it so **every** input, model output, and tool call passes through it — one enforcement point, no scattered `if`s.

### 2. Input and output rails (hand-rolled)

- Input rail: block moderation-flagged text; `transform` to redact PII (e.g. emails, card-like digit runs) before it reaches the model or your logs.
- Output rail: block harmful generations and redact PII that leaked in from retrieved/tool context.

### 3. The tool-call rail

- Before `read_file` runs, inspect the `path` argument: `allow` only paths that resolve inside an allowed root; `block` traversal (`../`) and absolute paths outside it.
- Confirm a blocked tool call **never executes** the tool.

### 4. Framework parity

- Re-implement the input/output moderation half with NeMo Guardrails **or** Guardrails AI. Note in `NOTES.md` what the framework gave you for free and what you still had to hand-roll (hint: the tool-call rail).

### 5. Fail closed and log

- Make the moderation backend raise/timeout on one call. Show the security-path rail **denies** rather than waving traffic through.
- Log every verdict (`rail`, `action`, `reason`) for later tuning.

## Starter guidance

```python
from dataclasses import dataclass
from pathlib import Path

@dataclass(frozen=True)
class Verdict:
    action: str          # "allow" | "block" | "transform"
    payload: str
    reason: str = ""

class GuardrailPolicy:
    def __init__(self, read_root: Path):
        self.read_root = read_root.resolve()

    def check_input(self, text: str) -> Verdict:
        raise NotImplementedError   # moderation + PII redaction

    def check_output(self, text: str) -> Verdict:
        raise NotImplementedError   # moderation + leak redaction

    def check_tool_call(self, tool: str, args: dict) -> Verdict:
        raise NotImplementedError   # arg-level allowlist; fail closed

def guarded_read_file(policy: GuardrailPolicy, path: str) -> str:
    v = policy.check_tool_call("read_file", {"path": path})
    if v.action == "block":
        return f"[blocked] {v.reason}"
    return Path(v.payload).read_text()
```

You do **not** need prompt-injection isolation (exercise-02) or sandboxing (exercise-03) here.

## Acceptance criteria

You can demonstrate that:

- One policy object guards all three surfaces; there is no input/output/tool path that bypasses it.
- A flagged input never reaches the model; a flagged output never reaches the user.
- A `read_file("../../etc/passwd")` style call is **blocked** and the tool never runs.
- When the moderation backend errors, the security-path rail **fails closed** (denies).
- Every verdict is logged with rail, action, and reason.

## Reflection

In `NOTES.md`:

1. Which rail caught the most in your testing — input, output, or tool-call? Why?
2. Give a false positive your input rail produced and how you'd tune it without opening a hole.
3. What did the framework handle that you'd have gotten wrong by hand, and what did it *not* cover?

## Stretch goals

- Add a `transform` that rewrites (not just blocks) a borderline output into a safe refusal.
- Swap your stub moderation for a real moderation model/API and compare false-positive rates.
- Add a second tool (`fetch_url`) and extend the tool-call rail with a host allowlist.
