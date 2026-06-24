# Chapter 1 — Input/Output Moderation and Per-Tool Guardrails

A guardrail is a deterministic check that the agent's traffic must pass through. It runs in *your* code, on its own schedule, regardless of what the model decides to do. Three places matter: the **input** the user (or a tool) feeds the model, the **output** the model produces, and each **tool call** the model wants to make.

```text
  user/tool input ──▶ [INPUT RAIL] ──▶ model ──▶ [OUTPUT RAIL] ──▶ user
                                          │
                                          ▼
                                   tool call request
                                          │
                                   [TOOL-CALL RAIL] ──▶ tool ──▶ result
```

Each rail is a function that returns a verdict: `allow`, `block`, or `transform` (redact, rewrite, or wrap). A blocked input never reaches the model; a blocked output never reaches the user; a blocked tool call never executes.

## What each rail checks

- **Input rail** — moderation (hate, self-harm, sexual, violence categories), banned-topic detection, and PII so you don't log a user's credit-card number. It also flags likely prompt injection ([Chapter 2](02-prompt-injection.md)).
- **Output rail** — moderation again (the model can generate harmful content even from benign input), PII leakage from retrieved context, and "did the model reveal the system prompt or a secret."
- **Tool-call rail** — the most security-critical. Before a tool runs, check the *arguments*: is this path inside the allowed directory, is this URL on the allowlist, is this SQL read-only? Permissions and sandboxing are [Chapter 3](03-tool-permissions.md).

A rail that only inspects the final text but not tool calls is a rail with a hole in it — most real damage happens through tools, not prose.

## Frameworks vs. hand-rolled

Two mature Python libraries package this:

- **NeMo Guardrails** (NVIDIA) — you define rails in a Colang-flavored config plus YAML, and the runtime intercepts input, output, and (with actions) tool calls. Strong for conversational "this topic is off-limits" flows.
- **Guardrails AI** — validator-centric: you compose `Guard` objects from validators (toxicity, PII, regex, competitor-mention, valid-JSON) that `validate()` a string and can `fix` or `reraise`.

Use a framework for the moderation/validation catalog you'd otherwise reimplement. Hand-roll the security-critical tool-call checks — they're short, and you want to read every line.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Verdict:
    action: str          # "allow" | "block" | "transform"
    payload: str         # original, or rewritten/redacted text
    reason: str = ""

def input_rail(text: str) -> Verdict:
    if moderation_flags(text):                       # call a moderation model/API
        return Verdict("block", text, "input flagged by moderation")
    return Verdict("transform", redact_pii(text), "pii redacted")
```

## Rules that keep rails honest

- **Fail closed on the security path.** If the moderation API times out, a tool-call rail should *deny*, not wave the call through. Availability is not worth a breach.
- **One enforcement point.** Route every input, output, and tool call through the same policy object. Scattered `if` checks are how a code path ends up unguarded.
- **Log every verdict.** Record what was blocked and why — you need this to tune false positives and to prove a control fired ([mod-205](../mod-205-evaluation-observability/README.md)).
- **Rails are layered, not redundant.** Input moderation does not make output moderation optional; injection defenses do not make tool allowlists optional. Each closes a different hole.

## Key takeaways

- A guardrail is code that runs regardless of the model — input rail, output rail, tool-call rail.
- The tool-call rail is the security-critical one; text moderation alone leaves the dangerous path open.
- Use NeMo Guardrails or Guardrails AI for the moderation/validation catalog; hand-roll tool-call checks.
- Fail closed, enforce at one point, and log every verdict.
